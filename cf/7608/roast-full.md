# Design pre-review: Add pg_stat_progress overview view

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7608](https://commitfest.postgresql.org/patch/7608/) (Needs review); authors: Josef Šimánek (retro)
- Thread: [\[PATCH v1\] Add pg_stat_progress overview view](https://www.postgresql.org/message-id/flat/CAFp7QwqYGVqF7LO6fDTthpaskRYNPaVXSPXKRQOrYT%3DXMe8kpw%40mail.gmail.com), started 2026-11-07
- Messages read: 1 of 1
- Patch prose read: v1-0001-Add-pg_stat_progress-overview-view.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The decisive objection is demand-1. Message #1 names no user, tool or report that needs the overview, and a user-defined `union all` over the existing progress views returns the same rows. The cost of the view is low, so this is "not now, show demand" and no rejection. The verdict rests on absence of evidence in a one-message thread, so confidence is low.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code I could not check how the view lists the command families, whether the new TAP test catches a missing command type, or whether the view keeps the existing filtering of other users' details.

## The proposal, restated

**Problem.** Finding active progress reports means querying each pg_stat_progress_* view separately. The patch targets users who monitor cluster operations and want to find all of them in one place.

**Proposed design.** A new system view, pg_stat_progress, has one row per backend reporting command progress, across all databases, with pid, command and progress_view. It is defined in system_views.sql on top of pg_stat_get_progress_info() and the existing progress permissions, with no backend changes. The pg_stat_progress_cluster compatibility view adds no extra row for REPACK, CLUSTER or VACUUM FULL.

**User-visible surface.**
- New system view pg_stat_progress with columns pid (integer), command (text) and progress_view (regclass).
- command values: ANALYZE, VACUUM, CREATE INDEX, BASEBACKUP, COPY, REPACK, DATACHECKSUMS; REINDEX shows as CREATE INDEX, CLUSTER and VACUUM FULL as REPACK.
- All three columns are visible to all users; details follow the permission rules of the detailed views.
- Catalog version bump.

**What the author claims.**
- The view needs no backend changes and reuses existing progress infrastructure.
- pid, command and progress_view are the minimal common fields; datid and datname are left out because base backups cover the whole cluster.
- Build, regression, isolation and new TAP tests passed locally.
- The longer-term aim is easier discovery of cluster operations and progress reporting for more commands.

**Thread state.** Only the first message is visible, and nobody has replied. The author asks for feedback on usefulness and columns; CFbot reports the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-07; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No demand shown beyond the author's wish for one overview

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

The view is added, and every later progress command must update its definition and docs. Monitoring tools already query the detailed pg_stat_progress_* views they care about, so few of them switch to it. The project carries the upkeep for a view that saves users one union query.

Message #1 gives the problem as an inconvenience and cites no field report, -general thread or tool that asked for this, and no other database's equivalent. A user can write the same three-column result today as a union over the seven detailed views, or over pg_stat_get_progress_info() per command, in a user view. The thread has no replies yet, so nobody else has said they need it. The cost is small, and the benefit is smaller still.

Thread: [#1 (Josef Šimánek, 2026-11-07)](https://www.postgresql.org/message-id/CAFp7QwqYGVqF7LO6fDTthpaskRYNPaVXSPXKRQOrYT%3DXMe8kpw%40mail.gmail.com)

**Question:** Which monitoring tools or users need this overview today, and why does a user-defined union of the existing progress views not serve them?

Counter-check: held. Message #1 names no user, tool or report asking for the overview, and the thread has no replies. The existing views already show pid for backends in all databases, so a user-defined union returns the same rows.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds the system view pg_stat_progress in system_views.sql and bumps catversion. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The documentation says pid, command and progress_view are visible to all users. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The view lists each command family by hand, so each new progress command must also edit it. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A user-defined `union all` over the existing pg_stat_progress_* views returns the same pid and command list. |

## 5. Questions the author must answer before anyone reviews code

1. Which monitoring tools or users need this overview today, and why does a user-defined `union all` over the existing pg_stat_progress_* views not serve them?
2. Why does a join of pg_stat_activity with the detailed progress views not already meet the need?
3. What makes the build or the regression tests fail when a new progress command type is added but pg_stat_progress is not updated?
4. Why does `pg_stat_progress.command` hold family names when `command` in the detailed progress views holds the command the user ran?
5. What use case needs progress_view as regclass when the command name already identifies the detailed view?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the progress reporting code and the statistics views in system_views.sql would have to own it. Nobody has replied in the thread.
- **Liability:** The owner signs for a permanent system view whose command list and documentation must change with each new progress command. Monitoring tools will depend on its column names and values.
- **Author follow-up:** Message #1 states a longer-term aim to extend progress reporting to more commands, which suggests continued work. CFbot reports the patch needs a rebase and no rebase is visible yet.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A report from a monitoring tool author or from users who need one list of all progress-reporting backends, with a reason why a user-defined union of the existing views is not enough. Support from other list members in the thread would also count.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** unclear
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 13 KiB, with one SQL view, documentation and a TAP test, and no backend changes, so review and upkeep are cheap. Message #1 shows no user or tool that needs it, so it can wait until demand is shown.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) The command column means something different from the same column in the detailed views. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) New progress commands must also update the overview view. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-2` New view instead of a progress column in pg_stat_activity. Refuted (no failure scenario): Progress reporting already lives in separate `pg_stat_progress_*` views that users join to `pg_stat_activity` on pid, so the patch follows existing practice. The name conflict depends on a later proposal that is not in the material, and a view can gain columns later.
- `already-exists-1` A union of the existing progress views already gives this list. Refuted (not specific): Duplicate of demand-1, which makes the same `union all` point and also covers the missing evidence of demand.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P2] proposed a new checkpoint progress view, so it is a different idea. Its recorded reason was that the view had fields of little use for monitoring, and this three-column view does not share that problem. The thread has one message and points to no earlier discussion of the same idea.
- standards-compat: 2 objection(s). The pg_dump round-trip, GUCs, grammar and behaviour for existing queries are not affected, because the patch adds only a system view.
- replication-ondisk: 0 objection(s). The proposal adds only a SQL view in system_views.sql and bumps catversion, with no backend, WAL, on-disk, wire protocol or ABI change. The view's catalog footprint is one more pg_rewrite entry, which initdb creates and pg_upgrade does not carry over. Keeping the view in step with new progress commands is a maintenance question for other angles.
- security: 0 objection(s). The view exposes nothing new to other users, because the detailed pg_stat_progress_* views already show pid and database to everyone and a row's presence shows the command. The view is SQL over pg_stat_get_progress_info() with no new privilege, function or DoS path, though without the code I could not check that it keeps the existing per-backend NULL filtering.
- maintenance-fit: 1 objection(s). From the maintenance angle there is one objection. The view adds no hot-path cost and no backend changes, and its cost falls only on users who query it.
- already-exists: 1 objection(s). One objection only. Documented views meet the need with modest effort, so the open point is whether the convenience is worth a permanent view.
- demand: 1 objection(s). One demand objection only. The cost of the view is low, so this reads as "not now, show demand" rather than a rejection.

### Precedents retrieved

- P1: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3545/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P2: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3873/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P3: [Add index scan progress to pg_stat_progress_vacuum](https://commitfest.postgresql.org/patch/3456/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P4: [Progress reporting for COPY command](https://commitfest.postgresql.org/patch/2607/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P5: [CLUSTER command progress monitor](https://commitfest.postgresql.org/patch/1268/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P6: [pg_stat_activity: avoid showing state=active with wait_event=ClientRead](https://commitfest.postgresql.org/patch/3760/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P7: [System username in pg_stat_activity](https://commitfest.postgresql.org/patch/4769/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P8: [ANALYZE command progress checker](https://commitfest.postgresql.org/patch/1053/) (cf, Returned with feedback, 2017-04-05) - judged not relevant; reason summarised from its thread
- P9: [Enhance file_fdw to report processed and skipped tuples in COPY progress](https://commitfest.postgresql.org/patch/5282/) (cf, Withdrawn, 2025-03-09) - judged not relevant; reason summarised from its thread
- P10: [ANALYZE command progress checker](https://commitfest.postgresql.org/patch/1104/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P11: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P12: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P13: [pg_stat_activity: make slow/hanging authentication more visible](https://commitfest.postgresql.org/patch/4974/) (cf, Withdrawn, 2025-03-21) - judged not relevant
- P14: [Add a statistics view to track usage of deprecated features](https://commitfest.postgresql.org/patch/6948/) (cf, Withdrawn, 2026-07-03) - judged not relevant
- P15: [Addition of authenticated ID to pg_stat_activity](https://commitfest.postgresql.org/patch/3095/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P16: [pg_stat_walwrites system view](https://commitfest.postgresql.org/patch/1003/) (cf, Returned with feedback, 2017-11-27) - judged not relevant
- P17: [PGDOCS - inconsistent linkends to "monitoring" views](https://commitfest.postgresql.org/patch/4602/) (cf, Rejected, 2023-11-30) - judged not relevant
- P18: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant
- P19: [pg_stat_statements: Add gc_count and query_file_size to pgss_info](https://commitfest.postgresql.org/patch/6554/) (cf, Returned with feedback, 2026-03-25) - judged not relevant
- P20: [Prevent incorrect entries in pg_stat_activity.](https://commitfest.postgresql.org/patch/5105/) (cf, Returned with feedback, 2025-10-17) - judged not relevant
- P21: [Display individual query in pg_stat_activity](https://commitfest.postgresql.org/patch/2656/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P22: [Waits monitoring](https://commitfest.postgresql.org/patch/297/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P23: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P24: [\[PATCH\]Refactor and unify expression construction functions in makefuncs.c](https://commitfest.postgresql.org/patch/6823/) (cf, Returned with feedback, 2026-06-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 139 s; cost reported by the CLI: $1.34
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:05:44.531Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
