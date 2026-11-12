# Design pre-review: pg_resetwal: add --cluster-state option

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7394](https://commitfest.postgresql.org/patch/7394/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [pg_resetwal: add --cluster-state option](https://www.postgresql.org/message-id/flat/CAGRkXqRH%3DxzB7svv-xFRF%3D6Lexo8tX0T4XX207uPAwuLys-Kcw%40mail.gmail.com), started 2026-10-05
- Messages read: 5 of 5
- Patch prose read: v2-0001-pg_resetwal-Add-cluster-state-option.patch, v2-0002-Test-pg_resetwal-cluster-state.patch, v1-0001-pg_resetwal-Add-cluster-state-option.patch, v1-0002-Test-pg_resetwal-cluster-state.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The proposal stops at level 1b on demand. Message #1 gives one use, emptying unlogged tables after a reset, and its question about other uses has no answer in the thread. A start, an immediate stop and a second start already reach that result (already-exists-1). Four of the five non-default values have no stated use (demand-2).

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how startup treats each hand-set state or how the option interacts with other pg_resetwal options. Messages #4 and #5 were available only through the brief and the objections.

## The proposal, restated

**Problem.** pg_resetwal always writes the shut-down state to pg_control, so the next start skips crash recovery. After a reset of a crashed cluster, unlogged tables keep whatever was on disk. The only use case the author can show is getting unlogged tables reset, and he asks Michael what other use he had in mind.

**Proposed design.** A new --cluster-state option tells pg_resetwal which state to store in pg_control instead of shut-down. Any other state makes the next start run crash recovery with no WAL to replay, so the other recovery steps still run. DB_STARTUP is excluded because the server refuses to start with it, and --dry-run shows the current and new state.

**User-visible surface.**
- pg_resetwal option --cluster-state=state, with values shut-down (default), shut-down-in-recovery, shutting-down, in-crash-recovery, in-archive-recovery, in-production.
- pg_resetwal --help lists the accepted --cluster-state values.
- pg_resetwal --dry-run shows the stored state under "Current pg_control values" and the new one under "Values to be changed".
- The documentation's claim that a reset of a cleanly shut down cluster leaves contents unchanged no longer holds for non-default states.

**What the author claims.**
- With a non-shut-down state, the next start runs crash recovery and empties unlogged tables, shown with a 1000-row table going to 0.
- A wrong state can be fixed by running pg_resetwal -f again, and the second reset loses nothing because the WAL is already gone.
- A mode that changes only the state without a reset would be a different tool, because setting shut-down on a crashed cluster would skip needed recovery.
- Michael Paquier suggested the option and David asked for a separate thread.

**Thread state.** Sho Ito tested v1 and v2 and confirmed they work; Chao Li found no code issues but asked for a state-only mode, which the author declined. The use case is still open, since Michael has not answered what he had in mind.

## 2. Prior attempts

### Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file (related)

- **Link:** [https://commitfest.postgresql.org/patch/3442/](https://commitfest.postgresql.org/patch/3442/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CALj2ACVvy+06hv7HocOFAHEBZWxLbc-t_61axdCUq9NaR2-c0Q@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALj2ACVvy+06hv7HocOFAHEBZWxLbc-t_61axdCUq9NaR2-c0Q@mail.gmail.com)
- **Date:** decided 2022-02-27; started 2021-12-06
- **Outcome:** Withdrawn (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier argued that the current state is not incorrect, because an end-of-recovery checkpoint is a shutdown checkpoint and the code has relied on that for about 10 years. He said the change was not checked against other assumptions, such as unlogged relation reset needing a clean shutdown, and that it leaves an invalid minimum recovery point in DB_IN_ARCHIVE_RECOVERY. He added that it would make back-patching harder in a sensitive area, and the entry was closed after the thread went quiet.
- **Reason (quote):** "This patch would bring some extra visibility for something that's not incorrect either on HEAD, as end-of-recovery checkpoints are the same things as shutdown checkpoints." ([message](https://www.postgresql.org/message-id/YhmaUxDMemv5hmba%40paquier.xyz))
- **What is different this time:** P5 changed which state the server writes during the end-of-recovery checkpoint. This patch lets a user write any state offline, so startup's dependence on the stored state matters for every accepted value.


## 3. Objections a reviewer is likely to raise

### 1. Existing idioms already reset unlogged tables after pg_resetwal

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A DBA resets a crashed cluster and wants unlogged tables emptied. Today they can start the server with listen_addresses = '', run pg_ctl stop -m immediate, and start again, which runs crash recovery and resets unlogged relations. The project would then keep a six-value option, docs and tests that only save this extra restart.

Message #1 shows one use case, emptying unlogged tables, and the author asks Michael for another, which has not arrived. An immediate stop already leaves the state in-production, so the next start runs crash recovery, the same path the option reaches. Running TRUNCATE on each unlogged table is a second existing route. The existing ways cost more steps, but they do the job, so the option adds convenience and no new capability.

**Precedents.**
- [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (Rejected, 2022-01-17); thread: [link](https://www.postgresql.org/message-id/flat/CALj2ACW0rV5gWK8A3m6_X62qH+Vfaq5hznC=i0R5Wojt5+yhyw@mail.gmail.com)

Thread: [#1 (shihao zhong, 2026-10-05)](https://www.postgresql.org/message-id/CAGRkXqRH%3DxzB7svv-xFRF%3D6Lexo8tX0T4XX207uPAwuLys-Kcw%40mail.gmail.com), [#4 (shihao zhong, 2026-10-09)](https://www.postgresql.org/message-id/CAGRkXqSPa3hzHgXLXgGvKEiWFCeTq5UoPTrxYQLVYpAXu0smCw%40mail.gmail.com)

**Question:** What does --cluster-state do that pg_resetwal followed by a start and pg_ctl stop -m immediate cannot, and who needs it?

Counter-check: held. A start, an immediate stop and a second start do reach crash recovery and reset unlogged tables, and the thread shows no other use. [P12] concerned a separate tool, so it should not be used as support here.

### 2. Five of the six state values have no stated use

fixable · level 1b, rubric item 5 · desirability, maintenance burden ("who maintains this in 10 years") · `demand-2`

A DBA reads the docs and picks in-archive-recovery or shut-down-in-recovery on a standby, because the docs list them as valid. Startup does something the docs do not describe, and nobody in the thread can say what the intended use was. Each new DBState value then needs a matching option value with no user behind it.

The one example in message #1 needs only some state other than shut-down, and in-production gives that. The patch still exposes all six values, and the v2 tests cover only in-production and a bad value. Each extra value adds documented behaviour that the project must explain and keep with no demand to justify it.

Thread: [#1 (shihao zhong, 2026-10-05)](https://www.postgresql.org/message-id/CAGRkXqRH%3DxzB7svv-xFRF%3D6Lexo8tX0T4XX207uPAwuLys-Kcw%40mail.gmail.com), [#4 (shihao zhong, 2026-10-09)](https://www.postgresql.org/message-id/CAGRkXqSPa3hzHgXLXgGvKEiWFCeTq5UoPTrxYQLVYpAXu0smCw%40mail.gmail.com), [#5 (Sho Ito, 2026-10-10)](https://www.postgresql.org/message-id/CAG7L5%2B7q7-Gq5OEWaWYx7%2BeP7Qej88k9F7BP7c7u-C2_w%3D3w6g%40mail.gmail.com)

**Question:** What concrete case needs each of the five values other than shut-down, and would a single "force crash recovery" switch cover the known need?

Counter-check: held. Message #1 states that the unlogged-table reset is the only known use, and in-production covers it. The thread gives no case for the other four non-default values, and the question to Michael has no answer.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The option writes a user-chosen value into the state field of pg_control; the file format does not change. |
| WAL / replication / standby | **yes** | A non-default state makes the next start run crash recovery, and two values name standby and archive recovery states. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Six documented option values map one-to-one onto the internal DBState enum and must stay valid across releases. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A start, pg_ctl stop -m immediate and a second start already empty unlogged tables after a reset. |

## 5. Questions the author must answer before anyone reviews code

1. Which use case, other than emptying unlogged tables, needs pg_resetwal to store a state other than shut-down?
2. What does --cluster-state do that pg_resetwal followed by a start, pg_ctl stop -m immediate and a second start cannot, and who needs it?
3. Which concrete case needs each of shut-down-in-recovery, shutting-down, in-crash-recovery and in-archive-recovery, or would a single switch that forces crash recovery be enough?
4. What does the next start do differently for each of the five non-default states after a reset?
5. How do in-archive-recovery and shut-down-in-recovery behave on a standby, or with standby.signal or recovery settings present?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the startup and recovery code and how it reads the pg_control state. None has posted in the visible thread.
- **Liability:** The owner signs for six documented state names tied to DBState, and for what startup does with each hand-set state, in every later release.
- **Author follow-up:** The author posted v2 within days of the reviews and answered both reviewers, which suggests he stays engaged.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A stated use case that the start, immediate stop and start sequence cannot serve, such as the motivation behind the original suggestion or a field report. For the extra values, one concrete case per state, or a reduction to a single switch that forces crash recovery.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is small and confined to pg_resetwal, but it serves one rare post-reset case and exposes six enum values where one is used. A single switch that forces crash recovery, with a stated use case, would be cheaper to carry.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Option exposes the internal DBState enum as a permanent user contract. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Six raw pg_control states exposed where only forced crash recovery is shown. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Six exposed states rest on startup assumptions nobody has checked. Refuted (not specific): Duplicate of replication-ondisk-1, which makes the same point about unchecked startup behaviour for each state and also names the single-switch alternative.
- `standards-compat-2` Dry-run state names differ from pg_controldata's spelling of the same field. Refuted (no failure scenario): The dry-run state lines are new, so no existing script matches them, and [P1] concerns changing existing output strings. Message #4 gives a reason for the spelling: the lines use the names the option takes.
- `maintenance-fit-1` One shown use case does not justify six user-settable states. Refuted (not specific): Duplicate of demand-2, which makes the same point without relying on [P12]. [P12] weighed a whole new tool, which is a larger cost than values of one option.
- `maintenance-fit-2` Internal DBState enum becomes a stable command-line interface. Refuted (not specific): Duplicate of standards-compat-1 for the enum-as-interface point and of replication-ondisk-1 for the startup assumptions. Its own scenario, a "damaged recovery", names no concrete case.
- `demand-1` No user need shown beyond one unlogged-table case. Refuted (not specific): Duplicate of already-exists-1, which states the same lack of shown need and describes the existing alternatives more concretely.

### Review angles

- prior-attempts: 1 objection(s). The proposal changes no existing pg_resetwal output, so [P1] does not apply. [P11] lapsed from inactivity. Message #1 points to an earlier suggestion by Michael, but the thread does not say what it was meant to solve.
- standards-compat: 2 objection(s). Two objections from the standards and compatibility angle. Existing users are unaffected, because the default stays shut-down.
- replication-ondisk: 1 objection(s). The only persistent state touched is the state field of pg_control. WAL format, catalogs, wire protocol and ABI do not change. The new dry-run line adds output without changing existing strings, so the pg_upgrade parsing concern in [P1] does not apply.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. pg_resetwal runs offline as the owner of the data directory, who can already rewrite pg_control or delete unlogged relation files directly, so --cluster-state gives no user an ability they lack today.
- maintenance-fit: 2 objection(s). Two objections raised; the change is small and confined to pg_resetwal, so hot-path cost and external dependencies are not concerns.
- already-exists: 1 objection(s). One objection from this angle; no existing tool sets an arbitrary pg_control state, so only the demonstrated use case is covered.
- demand: 2 objection(s). Demand is the gap here. The option works as tested, but the thread shows one use case and an open question to the person who suggested it.

### Precedents retrieved

- P1: [Revert ill-considered change in pg_resetwal output.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=918de37652138da7f7e54c79351178367d60f501) (revert, Reverted, 2022-09-15) - used in the review
- P2: [Revert "pg_resetwal: Improve error with wrong/missing data directory"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5daa50f962bd06e49af690efc2a8a27fdd37fb7d) (revert, Reverted, 2023-09-29) - judged not relevant
- P3: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P4: [Add recovery to pg_control and remove backup_label](https://commitfest.postgresql.org/patch/4629/) (cf, Withdrawn, 2024-01-28) - judged not relevant; reason summarised from its thread
- P5: [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://commitfest.postgresql.org/patch/3442/) (cf, Withdrawn, 2022-02-27) - used in the review; reason summarised from its thread
- P6: [Query Jumbling for CALL and SET utility statements](https://commitfest.postgresql.org/patch/3864/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P7: [Add control-C support for some psql meta-commands](https://commitfest.postgresql.org/patch/3412/) (cf, Withdrawn, 2021-11-15) - judged not relevant; reason summarised from its thread
- P8: [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (cf, Withdrawn, 2026-05-01) - judged not relevant; reason summarised from its thread
- P9: [psql: Allow editing query results with \gedit](https://commitfest.postgresql.org/patch/4788/) (cf, Rejected, 2024-05-14) - judged not relevant; reason summarised from its thread
- P10: [Revert "graceful shutdown" changes for Windows.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29992a6a509b256efc4ac560a1586b51a64b2637) (revert, Reverted, 2022-03-22) - judged not relevant
- P11: [Detectable crashes and unlogged table resets](https://commitfest.postgresql.org/patch/3183/) (cf, Returned with feedback, 2022-03-07) - used in the review; reason summarised from its thread
- P12: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - used in the review; reason summarised from its thread
- P13: [Avoid possibly slow processing during postmaster abnormal or immediate shutdown](https://commitfest.postgresql.org/patch/816/) (cf, Rejected, 2016-11-18) - judged not relevant
- P14: [New \[relation\] options engine](https://commitfest.postgresql.org/patch/3536/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P15: [Add callback table access method to reset filenode when dropping relation](https://commitfest.postgresql.org/patch/3073/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P16: [PITR shutdown should not report error by pg_ctl](https://commitfest.postgresql.org/patch/4653/) (cf, Withdrawn, 2023-11-30) - judged not relevant
- P17: [New \[relation\] options engine](https://commitfest.postgresql.org/patch/4160/) (cf, Returned with feedback, 2023-11-11) - judged not relevant
- P18: [standardize integer parsing in front-end utilities](https://commitfest.postgresql.org/patch/2197/) (cf, Withdrawn, 2020-03-05) - judged not relevant
- P19: [fix and document CLUSTER privileges](https://commitfest.postgresql.org/patch/4054/) (cf, Returned with feedback, 2023-01-14) - judged not relevant
- P20: [pg_walcleaner - new tool to detect, archive and delete the unneeded wal files](https://commitfest.postgresql.org/patch/3667/) (cf, Withdrawn, 2022-08-16) - judged not relevant
- P21: [Unlogged relation copy is not fsync'd](https://commitfest.postgresql.org/patch/4517/) (cf, Returned with feedback, 2024-01-05) - judged not relevant
- P22: [CLUSTER command progress monitor](https://commitfest.postgresql.org/patch/1268/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P23: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P24: [Revert "Prevent panic during shutdown checkpoint"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=703f148e98ecb4b299fdad403fc5a1de51220714) (revert, Reverted, 2017-06-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 200 s; cost reported by the CLI: $1.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 95 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:50:09.803Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
