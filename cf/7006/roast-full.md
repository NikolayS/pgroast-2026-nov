# Design pre-review: Add bypass IO to pg_stat_io

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7006](https://commitfest.postgresql.org/patch/7006/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Relation bulk write facility](https://www.postgresql.org/message-id/flat/30e8f366-58b3-b239-c521-422122dd5150%40iki.fi), started 2023-09-19
- Messages read: 55 of 55
- Patch prose read: 0001-Relax-fsyncing-at-end-of-bulk-load-that-was-not-WAL-.patch, v1-0001-Introduce-a-new-bulk-loading-facility.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

No surviving objection questions the goal, and message #3 shows a committer asked for this use of bulk_write.c. The two surviving objections, maintenance-fit-1 and standards-compat-1, are fixable design points about what the "bypass" rows cover and what the uncounted cells show. Confidence is low because the pg_stat_io patch has no visible reply, no documentation hunk, and its message was not shown in full.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The pg_stat_io patch and its documentation were not read, so the valid combinations for the new context, the row count and the counting cost could not be checked. The list of direct smgr writers outside bulk_write.c also needs the code.

## The proposal, restated

**Problem.** Writes done through bulk_write.c by CREATE INDEX, CLUSTER, VACUUM FULL and ALTER TABLE SET TABLESPACE call smgrextend()/smgrwrite() directly and never show in pg_stat_io. A large index build can write gigabytes that monitoring users cannot see. The deferred fsync is counted only under the checkpointer, and smgrimmedsync() is not counted at all.

**Proposed design.** The patch counts writes and extends in one place, smgr_bulk_flush(), under a new IOCONTEXT_BYPASS context. Timing uses the same calls as the buffered paths, so the *_time columns work with track_io_timing. Bypass reads and the fsync path stay untracked in this first patch.

**User-visible surface.**
- New pg_stat_io io_context value "bypass", with rows for writes and extends from bulk operations.
- Existing *_time columns filled for the new context when track_io_timing is on.

**What the author claims.**
- pg_stat_io was designed with this in mind: io_object was added to prepare for bypass IO.
- bulk_write.c is the central point that earlier pg_stat_io discussion deferred to.
- Andres Freund suggested in 2023 using the bulk write facility to track such writes in pg_stat_io.
- The name "bypass" matches the docs and avoids confusion with the BAS_BULKWRITE strategy ring.
- Counting smgrimmedsync() becomes the next step once this patch lands, as a comment in mdimmedsync() anticipates.

**Thread state.** The thread is mostly the 2023-2024 history of the bulk write facility, now committed, and the AIX removal it led to. Andrey Borodin's July 2026 message with the pg_stat_io patch has no visible reply, and the CF entry needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2023-09-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Counting only in smgr_bulk_flush() leaves other direct smgr writers silently uncounted

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A future core change or a table AM extension writes relation pages with smgrextend() or smgrwrite() outside bulk_write.c. Its writes do not appear under "bypass", and no test or assertion notices. A DBA reads the bypass rows as the full picture of IO outside shared buffers and misses that load during an incident.

Message #55 puts the accounting in one bulk_write.c function, while bypass reads and smgrimmedsync() stay uncounted. The context name promises all IO that bypasses shared buffers, but the proposal covers only one caller. The earlier pg_stat_io discussion that message #55 cites pointed to smgr wrappers, which would cover every caller. The proposal does not say why the bulk_write layer is better, or what stops new smgr callers from skipping the counters.

Thread: [#3 (Andres Freund, 2023-11-19)](https://www.postgresql.org/message-id/20231119000416.fuyrbo6av7dytmf6%40awork3.anarazel.de), [#55 (Andrey Borodin, 2026-07-01)](https://www.postgresql.org/message-id/789656F9-C573-474B-8C44-56FCCC16A8D3%40yandex-team.ru)

**Question:** Which smgrextend()/smgrwrite() callers stay outside bulk_write.c today, and how will future callers be counted or flagged instead of being missed silently?

Counter-check: held. Andres Freund suggested this counting point in message #3, which supports the placement. But the name "bypass" suggests full coverage, and the proposal does not list the smgrextend()/smgrwrite() callers outside bulk_write.c that stay uncounted.

### 2. Bypass rows would report reads and fsyncs that do happen as untracked

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An operator runs ALTER TABLE SET TABLESPACE, which reads through smgrread() in RelationCopyStorage, or a CREATE INDEX that hits smgrimmedsync(). The bypass row in pg_stat_io then shows NULL or 0 for reads and fsyncs, and dashboards trust it. After the planned follow-up patches the same columns fill in, so numbers jump between releases with no change in workload.

pg_stat_io uses NULL for combinations that cannot occur and 0 for counted-but-none. Message #55 says bypass reads and smgrimmedsync() do occur but stay uncounted for now, so either value is wrong for this release. The proposal shows no documentation hunk that defines the valid backend type, object and operation combinations for the new context. Changing what a released pg_stat_io row means in a later release is a cost that monitoring tools would have to handle.

Thread: [#55 (Andrey Borodin, 2026-07-01)](https://www.postgresql.org/message-id/789656F9-C573-474B-8C44-56FCCC16A8D3%40yandex-team.ru)

**Question:** What will the bypass row show for reads and fsyncs in this release, and how will the documentation keep that meaning stable once later patches count them?

Counter-check: held. Message #55 says bypass reads and smgrimmedsync() happen but stay uncounted, and the proposal has no documentation hunk saying what those cells show. Counters that fill in later are normal for pg_stat_io, so only the missing cell definition stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | A new IOContext value enlarges the cumulative statistics arrays and changes the enum that extensions compile against. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The new io_context value "bypass" becomes a released pg_stat_io meaning that later read and fsync patches must keep stable. |
| performance on the hot path | **yes** | The patch adds counting and optional timing in smgr_bulk_flush(), once per batched flush, with no measurement shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which smgrextend() and smgrwrite() callers stay outside bulk_write.c today, and will their writes appear under "bypass" or stay uncounted?
2. What will the bypass rows show for reads and fsyncs in this release: NULL, 0, or no such column value?
3. How will the documentation define the bypass rows so that their meaning stays stable once later patches count reads and smgrimmedsync()?
4. How many pg_stat_io rows does the new context add, and for which backend type and object combinations?
5. What is the measured overhead of counting and timing in smgr_bulk_flush() during a large CREATE INDEX with track_io_timing on?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows pg_stat_io and the smgr and bulk_write.c code would have to own it. Committers took part in the 2023-2024 thread, but none has replied to the July 2026 patch.
- **Liability:** The owner signs for the meaning of a new io_context value in a released monitoring view, and for keeping it consistent as follow-up patches add reads and fsyncs.
- **Author follow-up:** The author moved the entry to the next CommitFest in October 2026, but the rebase that CFbot asked for in July 2026 is not visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The change is additive, sits in one function, and helps users who monitor IO from index builds and table rewrites. It has waited since July 2026 with no reply and needs a rebase, so it needs a reviewer who knows pg_stat_io.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Using io_context for bypass departs from what io_context already describes. Refuted (wrong facts): These writes are not counted under 'vacuum' or 'bulkwrite' today, so no existing split is lost. The object written is still a relation, and message #55 cites the mdimmedsync() comment that asks for a separate IOContext for such IO.

### Review angles

- prior-attempts: 0 objection(s). No precedent records a rejection of tracking bypass writes in pg_stat_io. [P2] concerned the unbuffered write API and was returned because callers chose between the buffer manager and the new API themselves; the committed bulk_write.c in message #11 hides that choice, so that reason does not apply here. Message #5 refers to the earlier thread [P2] covers, and message #55 to pg_stat_io design discussions that deferred this work rather than rejecting it.
- standards-compat: 2 objection(s). The change is additive and needs no new GUC, syntax or pg_dump support. The objections above concern only how the new rows are defined.
- replication-ondisk: 0 objection(s). The proposal only adds counters and an io_context value in shared-memory statistics, with no WAL, catalog, relation-file, wire-protocol or replication change; the nbtree VACUUM bookkeeping invariant does not apply. A new IOContext enlarges the cumulative statistics arrays, so the stats file format ID and the extension ABI change on master only, which is routine for a new major version.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It adds a new io_context value to pg_stat_io, which already exposes the same kind of cluster-wide, per-backend-type counters to every role, with no relation names, users or query texts. It adds no new function, privilege, setting or code path run with elevated rights. Without the code I could not check how many rows the new context adds or what counting and timing cost, but neither question is about security.
- maintenance-fit: 1 objection(s). Hot-path cost looks small: counting happens once per batched flush, and timing follows the existing track_io_timing rules. No committer has replied to the July 2026 patch, which affects committability but not the merits. The nbtree VACUUM invariant does not apply to this proposal.
- already-exists: 0 objection(s). No existing core feature counts data writes from bulk_write.c: pg_stat_io misses them and pg_stat_wal counts only their WAL. The patch extends pg_stat_io instead of adding a second mechanism, and message #3 endorsed it. OS-level per-process counters from external tools do not attribute writes by context or object, so they do not meet the same need.
- demand: 0 objection(s). Demand for this is shown, though no field report is quoted. Andres Freund named this use of bulk_write.c in message #3, and message #55 cites the pg_stat_io design and an md.c comment that both expected it. The gap is concrete: a CREATE INDEX writes gigabytes that pg_stat_io does not show. The new counters and their small overhead affect only bulk operations. Overhead numbers and the number of new pg_stat_io rows are still missing, and other panel angles cover those.

### Precedents retrieved

- P1: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P2: [Avoid smgrimmedsync() during index build and add unbuffered IO API](https://commitfest.postgresql.org/patch/3508/) (cf, Returned with feedback, 2022-08-02) - used in the review; reason summarised from its thread
- P3: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant; reason summarised from its thread
- P4: [Allow cluster owner to bypass authentication](https://commitfest.postgresql.org/patch/2244/) (cf, Returned with feedback, 2020-04-06) - judged not relevant; reason summarised from its thread
- P5: [Revert "Use "transient" files for blind writes, take 2".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9bacf0e3734f0b5dc821abf7cb312377876e22d3) (revert, Reverted, 2012-10-17) - judged not relevant
- P6: [pg_stat_database update stats_reset only by pg_stat_reset](https://commitfest.postgresql.org/patch/2116/) (cf, Withdrawn, 2019-07-19) - judged not relevant; reason summarised from its thread
- P7: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P8: [Track number of WAL files ready to be archived in pg_stat_archiver](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Statistics_Collector) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P9: [pg_stat_statements with plans (v02)](https://commitfest.postgresql.org/patch/1470/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P10: [Temporary file access API](https://commitfest.postgresql.org/patch/3722/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P11: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant
- P12: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P13: [Postgres picks suboptimal index after building extended statistics](https://commitfest.postgresql.org/patch/3245/) (cf, Withdrawn, 2022-11-24) - judged not relevant
- P14: [pgrename_temp() – concurrent rename of temporary files on Windows](https://commitfest.postgresql.org/patch/2230/) (cf, Returned with feedback, 2020-04-05) - judged not relevant
- P15: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P16: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P17: [Add tracking of backend memory allocated to pg_stat_activity](https://commitfest.postgresql.org/patch/3865/) (cf, Withdrawn, 2023-01-05) - judged not relevant
- P18: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant
- P19: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [Allow index AMs to calculate custom statistics / BRIN](https://commitfest.postgresql.org/patch/3952/) (cf, Returned with feedback, 2022-12-17) - judged not relevant
- P21: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - judged not relevant
- P22: [Revert "Temporarily instrument postgres_fdw test to look for statistics changes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=04e7ecadf64d18f67ca1d0632d8ab71f120ca5e3) (revert, Reverted, 2018-03-08) - judged not relevant
- P23: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P24: [Speeding up GIST index builds using popcount](https://commitfest.postgresql.org/patch/3023/) (cf, Returned with feedback, 2021-10-01) - judged not relevant

### Warnings

- review angle "maintenance-fit" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 99 s; cost reported by the CLI: $3.41
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:14:01.561Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
