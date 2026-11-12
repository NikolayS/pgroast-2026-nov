# Design pre-review: postgres_fdw: Use COPY to speed up batch inserts

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6137](https://commitfest.postgresql.org/patch/6137/) (Needs review); authors: Matheus Alcantara (matheus.alcantara)
- Thread: [postgres_fdw: Use COPY to speed up batch inserts](https://www.postgresql.org/message-id/flat/DDIZJ217OUDK.2R5WE4OGL5PTY%40gmail.com), started 2025-10-15
- Messages read: 37 of 37
- Patch prose read: v15-0001-Extract-CopyEscapeText-for-reuse-outside-COPY-TO.patch, v15-0002-postgres_fdw-Use-COPY-as-remote-SQL-when-possibl.patch, v1-0001-postgres_fdw-Use-COPY-to-speed-up-batch-inserts.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support from several reviewers, but demand-1 survives at level 1b. The only whole-file benchmark (message #36) shows v14 and v16 about twice as slow as HEAD, and only a variant that keeps COPY IN open was faster. Message #35 rejected that variant for connection sharing, and no reply reconciles the two. The benefit of the design as proposed is not yet shown, so design objections were not ranked.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 37 messages and no v16 patch text were read, so the benchmark and cancel reports rest on the brief's summary of message #36. The four surviving level 2 objections were not evaluated because level 1b fails first.

## The proposal, restated

**Problem.** postgres_fdw sends batches to the remote server as prepared INSERT statements, even when the user runs COPY FROM on a foreign table. The author says remote COPY is faster for large batch loads into postgres_fdw tables.

**Proposed design.** The latest version (v16) adds an optional FdwRoutine callback, ExecForeignBatchCopy, which CopyFrom() calls for each buffered batch when the target is a foreign table or foreign partition and batch_size > 1. postgres_fdw runs a self-contained COPY ... FROM STDIN in text format per call. It falls back to batch INSERT when the target has AFTER ROW triggers, and a refactor exposes CopyEscapeText() for reuse.

**User-visible surface.**
- New optional FDW API callback ExecForeignBatchCopy(estate, rinfo, slots, numSlots) in fdwapi.h, documented in the FDW docs.
- New public function CopyEscapeText() and macros CopySendDataBuf() and CopySendCharBuf() in commands/copy.h.
- postgres_fdw sends COPY ... FROM STDIN to the remote server, so remote logs and remote triggers that check current_query() see COPY.
- Earlier versions added the table and server options use_copy_for_insert, use_copy_for_batch_insert and batch_with_copy_threshold; v16 drops them.

**What the author claims.**
- COPY FROM into a foreign table with 100 rows runs about twice as fast at batch_size 100 and 1000 (v14 numbers).
- Self-contained COPY per batch lets foreign partitions share one cached connection and lets other queries run on it between batches.
- With batch_size = 1, per-row COPY is slower than INSERT, so the patch keeps row-by-row INSERT then.
- A dedicated callback could later support user-chosen COPY options and BeginForeignCopy/EndForeignCopy.

**Thread state.** Masahiko Sawada reviewed several versions and agreed off-list to the dedicated callback design; Kirill Reshke benchmarked v14 and v16, found them about 2 times slower than HEAD across VMs without connection-level COPY caching, and reported uncancellable COPY. The CommitFest entry needs a rebase and no one has replied to Kirill.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-10-15; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Only independent benchmark shows v16 slower than HEAD for COPY FROM

fixable · level 1b, rubric item 2 · desirability, performance on the hot path, ABI / extension API · `demand-1`

A user loading a large file with `COPY ft FROM` into a postgres_fdw table with batch_size set upgrades and expects a faster load. In Kirill's two-VM setup the load of 1M rows takes about twice as long as on HEAD. The user finds out on the first production load.

The author's speedup numbers (message #31) come from pgbench running COPY of 100 rows per transaction. Kirill measured COPY of whole files in message #36: v14 and v16 were slower than HEAD on local and cross-AZ VMs, and only his draft connection-level COPY caching made it faster. That caching keeps COPY IN open across batches, which message #35 rejected because of connection sharing. No reply to message #36 is visible, so the user benefit of the design as proposed is not shown.

Thread: [#31 ("Matheus Alcantara", 2026-04-07)](https://www.postgresql.org/message-id/DHN84NSGBYYO.123WDP7K8D21D%40gmail.com), [#35 (Matheus Alcantara, 2026-08-05)](https://www.postgresql.org/message-id/CAFY6G8ekRx2U9Bno7ft9fTkth2HX1rA9GKNVnJtqn%3D2X76Xcyw%40mail.gmail.com), [#36 (Kirill Reshke, 2026-09-09)](https://www.postgresql.org/message-id/CALdSSPho7cJBAtNnHJLpCz_zOAqA%3DhQz%3DK125zpSORKaW%3DJA5g%40mail.gmail.com)

**Question:** For a bulk COPY FROM of a large file across a real network, is v16 faster than HEAD, and if only with persistent COPY IN, how does that work with shared connections?

Counter-check: held. The goal has support in message #2, but the only whole-file measurements (message #36) show v14 and v16 slower than HEAD. The author's numbers use 100-row COPY in pgbench, and no reply addresses the difference.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v16 adds the optional FdwRoutine callback ExecForeignBatchCopy and exports CopyEscapeText() in commands/copy.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Remote COPY FROM treats row-level security, views and rules differently from the remote INSERT used today. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new public FDW callback and a COPY IN state that postgres_fdw abort and cancel code must handle. |
| performance on the hot path | **yes** | The patch changes how CopyFrom() flushes each batch to foreign tables, and message #36 measured it slower than HEAD. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. For COPY FROM of a large file between two hosts, is v16 faster than HEAD, and at which batch_size values?
2. If only a COPY IN kept open across batches beats HEAD, how does that work with foreign partitions that share one cached connection?
3. How does v16 end an open COPY IN on the remote connection when the user cancels or a row fails, and which test covers it?
4. What happens when the remote target is a view, has rules, or has row-level security for the mapped role, and how can users keep remote INSERT?
5. What can ExecForeignBatchCopy do that ExecForeignBatchInsert cannot, given a flag that says the caller is COPY FROM?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Masahiko Sawada reviewed several versions and agreed off-list to the callback design. The owner needs to know the FDW API, CopyFrom() batching and postgres_fdw connection cleanup.
- **Liability:** A public FDW callback that third-party FDWs may implement and that is hard to change later. Also the COPY IN state in postgres_fdw abort and cancel handling, for as long as postgres_fdw exists.
- **Author follow-up:** The author posted 16 versions and moved the entry forward on 2026-10-01. The entry has needed a rebase since 2026-09-09 and message #36 has no reply.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Users who bulk load into postgres_fdw tables or foreign partitions would gain, and a committer is engaged. But the per-batch COPY design measured slower than HEAD in message #36, and the callback set is not settled, so more code review should wait.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Remote COPY silently replaces INSERT, with different semantics on the remote side. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-2` (fixable, level 2) Abort cleanup does not handle a remote connection left in COPY IN. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) New FDW callback is shaped around a design measured slower than HEAD. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `already-exists-1` (fixable, level 2) New FDW callback duplicates ExecForeignBatchInsert for one caller. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `replication-ondisk-1` Remote COPY rejects targets that remote INSERT accepts today. Refuted (not specific): Duplicate of standards-compat-1, which covers the same remote view and RLS cases and also rules.
- `security-1` Remote COPY in progress cannot be cancelled during abort cleanup. Refuted (not specific): Duplicate of replication-ondisk-2. The privilege framing adds no separate failure, since the same hang is the whole scenario.
- `security-2` Remote tables with row-level security reject COPY FROM. Refuted (not specific): Duplicate of standards-compat-1, which already includes the remote RLS case and the missing opt-out.
- `maintenance-fit-2` postgres_fdw cleanup paths do not handle a connection left in COPY IN. Refuted (not specific): Duplicate of replication-ondisk-2. The added claim about future changes to connection.c names no concrete case beyond the same cancel hang.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P9] and [P10] add an FDW callback for TRUNCATE, a different idea, and their reasons (unfinished docs and tests, a stalled rebase) do not apply to this design. Inside the thread, message #35 reports that a dropped BeginForeignCopy/EndForeignCopy design failed on connection sharing. Kirill's results in message #36 point back toward that design, and the performance and design angles should weigh this.
- standards-compat: 1 objection(s). Other contract points looked acceptable from this angle. No new SQL syntax, GUC, catalog object or pg_dump state is added, and the batch cap of 1000 rows already exists for COPY into foreign tables.
- replication-ondisk: 2 objection(s). There is no on-disk, WAL or catalog state; v16 drops the new options. The new FdwRoutine member is optional and only matters for extensions on new major versions.
- security: 2 objection(s). Default privileges do not change: COPY FROM STDIN on the remote side needs the same INSERT privilege that remote INSERT needs. The two objections cover cancellation and remote RLS.
- maintenance-fit: 2 objection(s). Masahiko Sawada, a committer, reviewed most versions, so who maintains this is less of a concern. The CopyEscapeText refactor in 0001 removes duplicated code and can stand on its own.
- already-exists: 1 objection(s). Postgres has no existing way to make COPY FROM into a postgres_fdw table use remote COPY. Running COPY directly on the remote server does not cover tuple routing into foreign partitions. So the need itself is not already met.
- demand: 1 objection(s). Demand for faster bulk loads into postgres_fdw tables is shown: the idea comes from an earlier blog post, and several independent reviewers tested it. The open question is whether v16 delivers the benefit.

### Precedents retrieved

- P1: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P2: [Asynchronous execution on postgres-fdw](https://commitfest.postgresql.org/patch/291/) (cf, Rejected, 2015-08-10) - judged not relevant; reason summarised from its thread
- P3: [Async execution of postgres_fdw.](https://commitfest.postgresql.org/patch/32/) (cf, Rejected, 2015-05-09) - judged not relevant; reason summarised from its thread
- P4: [Push down more UPDATEs/DELETEs in postgres_fdw](https://commitfest.postgresql.org/patch/791/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P5: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [postgres_fdw super user checks](https://commitfest.postgresql.org/patch/824/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P7: [postgres_fdw: support parameterized foreign joins](https://commitfest.postgresql.org/patch/1042/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P8: [Adding Support for Copy callback functionality on COPY TO api](https://commitfest.postgresql.org/patch/2660/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P9: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - used in the review; reason summarised from its thread
- P10: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2499/) (cf, Returned with feedback, 2020-08-02) - used in the review; reason summarised from its thread
- P11: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant
- P12: [explain plans for foreign servers](https://commitfest.postgresql.org/patch/5525/) (cf, Returned with feedback, 2025-08-05) - judged not relevant
- P13: [Revert "Refactor CopyReadAttributes{CSV,Text}() to use a callback in COPY FROM"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=06bd311bce24083c76d9741ae89c98750aaf4b41) (revert, Reverted, 2024-02-14) - judged not relevant
- P14: [postgres_fdw: Add support for INSERT OVERRIDING clause](https://commitfest.postgresql.org/patch/1357/) (cf, Returned with feedback, 2017-12-28) - judged not relevant
- P15: [Asynchronous execution of ForeignScans](https://commitfest.postgresql.org/patch/751/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P16: [Revert "Refactor CopyAttributeOut{CSV,Text}() to use a callback in COPY TO"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aa8324b81fa4979ffdc6ccf81d560eac9446948) (revert, Reverted, 2024-02-06) - judged not relevant
- P17: [Enhance file_fdw to report processed and skipped tuples in COPY progress](https://commitfest.postgresql.org/patch/5282/) (cf, Withdrawn, 2025-03-09) - judged not relevant
- P18: [Allow COPY "text" to output a header and add header matching mode to COPY FROM](https://commitfest.postgresql.org/patch/2504/) (cf, Returned with feedback, 2020-09-29) - judged not relevant
- P19: [copy_srf (was COPY as a set returning function)](https://commitfest.postgresql.org/patch/869/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P20: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P21: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant
- P22: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/167/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P23: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant
- P24: [COPY RAW](https://commitfest.postgresql.org/patch/223/) (cf, Returned with feedback, 2016-04-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 159 s; cost reported by the CLI: $4.02
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 95 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:43:48.646Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
