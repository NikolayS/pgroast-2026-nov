# Design pre-review: pg_stat_io_histogram

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6433](https://commitfest.postgresql.org/patch/6433/) (Needs review); authors: Jakub Wartak (jakub.wartak.tomtom)
- Thread: [pg_stat_io_histogram](https://www.postgresql.org/message-id/flat/CAKZiRmwvE4uJLKTgPXeBA4m%2Bd4tTghayoefcaM9%3Dz3_S7i72GA%40mail.gmail.com), started 2026-01-26
- Messages read: 36 of 36
- Patch prose read: v15-0001-Add-pg_stat_io_histogram-view-to-provide-more-de.patch, v15-0002-Lower-pg_stat_io_histogram-private-backend-memor.patch, v15-0003-Lower-pg_stat_io_histogram-shared-memory-use-by-.patch, v1-0001-Add-pg_stat_io_histogram-view-to-provide-more-det.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

Nothing that survived questions the goal: several reviewers report demand, and the measured CPU cost is within about 1% of master. Three fixable design points remain: standards-compat-1 (existing backends and the checkpointer record no histograms after a reload), standards-compat-2 (no agreed bucket range or view name) and replication-ondisk-1 (the pgstat file layout depends on the tracked combinations). A committer who knows the pgstat code needs to rule on these and on the v10 to v15 memory design.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether the pgstat file load validates the histogram slot count, or whether histograms could be allocated lazily outside critical sections. Only 3 of 36 messages were read directly.

## The proposal, restated

**Problem.** Users report that Postgres is slow or stuck, often at COMMIT, when the cause lies in the I/O stack. Proving this today means correlating iostat, wait events and other sources by hand, which takes too long. The author wants DBAs and support engineers to spot I/O latency outliers, such as slow fsyncs, directly from Postgres.

**Proposed design.** The patch keeps a latency histogram of power-of-two buckets for each tracked backend type, object, context and I/O operation, filled where pg_stat_io already counts I/O time. Recording happens only with track_io_timing or track_wal_io_timing on, and only in backends started after the setting is on. Later patches cut memory by storing histograms only for valid combinations, using an offset array in backend memory and shared memory sized at startup.

**User-visible surface.**
- New view pg_stat_io_histogram with columns backend_type, object, context, io_type, bucket_latency_us (an int4range) and bucket_count.
- New set-returning function pg_stat_get_io_histogram() behind the view.
- track_io_timing and track_wal_io_timing now also turn on histogram collection, but only for newly started backends.
- Histogram data persists in the pgstat file across restarts; the file format changes, but the PGSTAT_FILE_FORMAT_ID bump is left to the committer.

**What the author claims.**
- CPU overhead is negligible; Ants Aasma's pgbench and seq scan runs show results within about 1% of master, and Tomas Vondra reports the same.
- Buckets run from 8us to about 131ms and separate page cache hits from device reads.
- The v10 to v15 design lowers per-backend memory below master's and adds about 50 KiB of shared memory, per the author's measurements in message #28.
- Logging slow I/O through a new log_* GUC is a worse alternative, because of threshold choice and log spam.

**Thread state.** Andres Freund, Ants Aasma and Tomas Vondra reviewed; Andres asked to store only usable histograms, and the author's v10 to v15 do this with no reply. Since May only the author has posted, with rebases and a self-review, and the patch has moved through four CommitFests.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-01-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Histogram collection ignores track_io_timing changes in running backends

fixable · level 2, rubric item 10 · new GUC, psql UX conventions · `standards-compat-1`

During an incident a DBA runs ALTER SYSTEM SET track_io_timing = on and reloads to find slow fsyncs. The checkpointer, walwriter and pooled client connections keep running, so they record no histogram data, but pg_stat_io still shows their I/O time. The DBA sees an empty or partial histogram and wrongly decides fsync is not the problem.

Message #24 and message #36 state that histograms are filled only in backends started after the setting is on, and that SET in a session does not enable collection. The same GUC therefore turns on timing in pg_stat_io at once but turns on histograms only after a restart of each process. Slow COMMIT and checkpointer fsync, the cases in message #1, come from long-lived processes, so they are the ones most likely to be missed. The docs note in message #36 describes the gap but does not justify giving one existing GUC two different meanings.

Thread: [#1 (Jakub Wartak, 2026-01-26)](https://www.postgresql.org/message-id/CAKZiRmwvE4uJLKTgPXeBA4m%2Bd4tTghayoefcaM9%3Dz3_S7i72GA%40mail.gmail.com), [#24 (Jakub Wartak, 2026-03-06)](https://www.postgresql.org/message-id/CAKZiRmz-KH2a_qcwGeFwsoajCsQEd8jXyFLx0B0yKgkK%3DLhBLA%40mail.gmail.com), [#36 (Jakub Wartak, 2026-09-14)](https://www.postgresql.org/message-id/CAKZiRmyuuToHXESA0pCQ1ZT6_auhAOHzV0Eb%2B5r2tRaTOtfrFw%40mail.gmail.com)

**Question:** How does a DBA get histogram data for the checkpointer and existing connections without a server restart, and why should track_io_timing act differently for pg_stat_io and pg_stat_io_histogram?

Counter-check: held. Messages #24 and #36 confirm that only newly started processes collect histograms, and the v15 docs note states the gap without justifying it. The checkpointer and pooled connections are long-lived, so the incident case in message #1 is the one affected.

### 2. Bucket edges and view name become a fixed contract before agreement

fixable · level 2, rubric item 6 · SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

Monitoring tools build alerts on rows such as bucket_latency_us = '[131072,)' or on the 8us first bucket. A later release moves the range to 64us through 1s, or to 32s, as reviewers suggested. Those alerts then match nothing or the wrong latency, and users find out only when an outage goes unreported.

The view exposes bucket edges as data values, but message #25 proposes starting at 64us, message #9 and message #29 want buckets up to tens of seconds, and the author keeps 8us to 131ms. Message #36 says the labels are approximate because buckets are computed in nanoseconds, so [8,16) really covers 8.192us to 16.384us. Message #26 also floats renaming the view to pg_stat_io_lat_histogram to leave room for size histograms. [P2] was set aside so that the meaning of pg_stat_io would not change between releases, and the same reasoning applies to these edges.

**Precedents.**
- [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (Returned with feedback, 2024-01-18); thread: [link](https://www.postgresql.org/message-id/flat/CAN55FZ3AiQ+ZMxUuXnBpd0Rrh1YhwJ5FudkHg=JU0P+-W8T4Vg@mail.gmail.com)

Thread: [#9 (Ants Aasma, 2026-02-11)](https://www.postgresql.org/message-id/CANwKhkM1FNS1Wmc56%2BaunXhaP_zjjO2YKzKTJRoVW0RsM4Of3w%40mail.gmail.com), [#25 (Tomas Vondra, 2026-03-16)](https://www.postgresql.org/message-id/4ffe367d-77ac-488a-8826-64efb78d6f78%40vondra.me), [#26 (Jakub Wartak, 2026-03-17)](https://www.postgresql.org/message-id/CAKZiRmy8qMeFprFqm0cyQQXtBRbP2_3TVNwxYOeedD-Yro-P%3DQ%40mail.gmail.com), [#29 (Jakub Wartak, 2026-03-19)](https://www.postgresql.org/message-id/CAKZiRmxTEAbrif9zwZ7H7%3DdmC4espPgxQYEbs7F%3DMNk%3D80sAjw%40mail.gmail.com), [#36 (Jakub Wartak, 2026-09-14)](https://www.postgresql.org/message-id/CAKZiRmyuuToHXESA0pCQ1ZT6_auhAOHzV0Eb%2B5r2tRaTOtfrFw%40mail.gmail.com)

**Question:** Which bucket range, exact edge labels and view name will be kept stable across releases, and how would a later change avoid breaking queries that filter on bucket_latency_us?

Counter-check: held. The thread shows no agreement on the bucket range (messages #9, #25), and the edges are exposed as data values. [P2] concerned op_bytes semantics for WAL and must not be used as support.

### 3. Persisted histogram layout depends on pgstat_tracks_io_* logic

fixable · level 2, rubric item 9 · on-disk format / pg_upgrade, backpatchability, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A later minor release backpatches a fix that makes pgstat_tracks_io_op() accept one more combination. The slot count and offsets change, but nobody bumps PGSTAT_FILE_FORMAT_ID. After the restart, the server loads the old pgstat file and shows fsync counts under the wrong backend type or operation, and no error is raised.

Today pg_stat_io stores a fixed [object][context][op] array, so changing which combinations are tracked does not change the file layout. Per v15-0003, the histogram slot count now comes from pgstat_io_get_sum_tracked() at startup, which ties the pgstat file layout to code that looks like pure policy. Message #36 adds safeguards against reading corrupted data but does not say whether a load checks the slot count or the offset map. Message #35 also leaves the format ID bump to the committer, so the coupling is not yet recorded anywhere.

Thread: [#31 (Jakub Wartak, 2026-05-08)](https://www.postgresql.org/message-id/CAKZiRmx3Z%2BfxunEc4CDGrpZVFwD4MAEDfGVM_v-CW-%3DxTLeF_Q%40mail.gmail.com), [#35 (Jakub Wartak, 2026-08-25)](https://www.postgresql.org/message-id/CAKZiRmyyUFLiF-jp15Z40VgHeB1SMYs4rc_efZmgW1B0z0fWag%40mail.gmail.com), [#36 (Jakub Wartak, 2026-09-14)](https://www.postgresql.org/message-id/CAKZiRmyuuToHXESA0pCQ1ZT6_auhAOHzV0Eb%2B5r2tRaTOtfrFw%40mail.gmail.com)

**Question:** Does loading the pgstat file reject or discard histograms when the stored slot count or offset map differs from the one the running server computes?

Counter-check: held. v15-0003 derives the slot count from the pgstat_tracks_io_* functions at startup, so the file layout now depends on them. Message #36 mentions read safeguards but does not say they compare the stored slot count or offset map.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Histograms persist in the pgstat file, so its format changes and a PGSTAT_FILE_FORMAT_ID bump is needed. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds the pg_stat_io_histogram view in system_views.sql and pg_stat_get_io_histogram() in pg_proc.dat. |
| ABI / extension API | **yes** | The patch changes the I/O stats structures in pgstat.h and pgstat_internal.h and adds to pg_bitutils.h. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: Bucket edges and view name become a fixed contract before agreement |
| new GUC | **yes** | No new GUC, but track_io_timing and track_wal_io_timing gain a second effect that applies only to newly started backends. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | see objection standards-compat-1: Histogram collection ignores track_io_timing changes in running backends |
| security / privileges | no |  |
| backpatchability | **yes** | A backpatched change to pgstat_tracks_io_op() would change the histogram slot count and so the pgstat file layout. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Bucket edges become user-visible values, and pgstat_io gets shared memory sized at startup, unlike other fixed stats kinds. |
| performance on the hot path | **yes** | Each timed I/O in pgstat_count_io_op_time() also computes a bucket and increments a counter when timing is on. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does a DBA get histogram data for the checkpointer and existing connections after turning on track_io_timing with a reload, without a server restart?
2. Which bucket range will stay fixed across releases, given the open proposals to start at 64us and to extend to tens of seconds?
3. Does loading the pgstat file discard histograms when the stored slot count or offset map differs from what the running server computes?
4. Will the view be named pg_stat_io_histogram or pg_stat_io_lat_histogram, given the idea of later size histograms?
5. Will bucket_latency_us show exact edges, given that buckets are computed in nanoseconds but labelled in microseconds?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund reviewed early versions and asked to store only usable histograms, but has not replied to v10 to v15. The owner must know the cumulative statistics code and the I/O accounting in pgstat_io.c.
- **Liability:** The owner signs for a new pgstat file layout tied to the tracked I/O combinations, for bucket edges that monitoring tools will depend on, and for extra work on the timed I/O path. All three last as long as the view exists.
- **Author follow-up:** The author has rebased through four CommitFests, posted v10 to v15 in answer to review, and posted a self-review with no outside reply since May.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Three reviewers support the goal and the cost falls mostly on users of the timing settings, but no reviewer has answered v10 to v15 since May. The three patches total about 70 KiB and are confined to the pgstat code, with a new view and a file format change to carry.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Histograms only start in backends spawned after the timing setting. Refuted (not specific): Duplicate of standards-compat-1, which also covers the checkpointer and other long-lived auxiliary processes.
- `maintenance-fit-2` Startup-sized shared memory makes pgstat_io a special case. Refuted (not specific): Duplicate of replication-ondisk-1 for the file layout risk. The startup-sized allocation follows Andres Freund's request in messages #16 and #27 to store only usable histograms, so the departure has a stated reason.
- `maintenance-fit-3` Bucket edges become user-visible API before the range is settled. Refuted (not specific): Duplicate of standards-compat-2, which states the same bucket range concern with more detail.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P15] and [P22] cover wait-event monitoring and record no reason, and the op_bytes dispute in [P2] does not apply to latency buckets. The thread cites no earlier attempt at I/O latency histograms.
- standards-compat: 2 objection(s). Two objections from the standards and compatibility angle; reset behaviour and pg_dump do not apply or are not described in the material.
- replication-ondisk: 1 objection(s). Apart from the pgstat file, the proposal adds no WAL records, no relation or catalog on-disk change beyond a view and a function, and no wire protocol change. Stats are not carried by pg_upgrade, so old clusters are not affected.
- security: 0 objection(s). No security objection found. The view aggregates cluster-wide counts by backend type, like pg_stat_io, so it exposes no per-user, per-relation or per-query data, and its memory is sized at startup. v15 uses INT4RANGEOID directly, which removes the search_path lookup of int4range raised in message #36.
- maintenance-fit: 3 objection(s). Hot-path CPU cost was measured by several reviewers and appears negligible, so no objection is raised on items 4 or 8.
- already-exists: 0 objection(s). No existing feature gives this result. pg_stat_io keeps only total read_time, write_time and fsync_time per backend type, object and context, so a few slow fsyncs vanish into the average. eBPF tools such as bpftrace (used in message #3) need root, which managed services do not grant, and they have no Postgres context. A hook plus a custom cumulative statistics kind could move the feature into an extension, but it would still add a call on the same I/O path. Three reviewers who know the area (messages #2, #9 and #25) supported core placement, so a committer would likely not send this objection.
- demand: 0 objection(s). The material shows real demand from several independent people. The author reports repeated support cases where Postgres looked slow at COMMIT and the cause was the I/O stack (message #1). Ants Aasma reports multiple diagnosis cases, including 20s to 60s latencies from multipathing faults, where a histogram would have helped (message #9). Andres Freund calls the feature interesting (message #2), and Tomas Vondra gives +1 from his EXPLAIN prefetch work (message #25). The cost falls mostly on users who turn on track_io_timing or track_wal_io_timing, and the later versions cut the remaining memory cost (message #28). The remaining disputes are about memory layout and bucket range, which other angles cover.

### Precedents retrieved

- P1: [Syncrep and improving latency due to WAL throttling](https://commitfest.postgresql.org/patch/5309/) (cf, Withdrawn, 2025-11-14) - judged not relevant; reason summarised from its thread
- P2: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - used in the review; reason summarised from its thread
- P3: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P4: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P5: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P6: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P7: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant; reason summarised from its thread
- P8: [Operation log for major operations](https://commitfest.postgresql.org/patch/4018/) (cf, Withdrawn, 2023-03-02) - judged not relevant; reason summarised from its thread
- P9: [WIP: Track relation size in shared memory](https://commitfest.postgresql.org/patch/2933/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P10: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P11: [pg_stat_activity: make slow/hanging authentication more visible](https://commitfest.postgresql.org/patch/4974/) (cf, Withdrawn, 2025-03-21) - judged not relevant
- P12: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant
- P13: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P14: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P15: [Add accumulated statistics for wait event](https://commitfest.postgresql.org/patch/2332/) (cf, Returned with feedback, 2020-08-02) - used in the review
- P16: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P17: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/2496/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P18: [pg_stat_activity: avoid showing state=active with wait_event=ClientRead](https://commitfest.postgresql.org/patch/3760/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P19: [history of vacuum / checkpoint runs](https://commitfest.postgresql.org/patch/5462/) (cf, Withdrawn, 2025-11-14) - judged not relevant
- P20: [PGDOCS - inconsistent linkends to "monitoring" views](https://commitfest.postgresql.org/patch/4602/) (cf, Rejected, 2023-11-30) - judged not relevant
- P21: [Improve management of SLRU statistics collection.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=81ca8686305c4c62d723ab224ad5c414f350a3a0) (revert, Partially reverted, 2020-05-13) - judged not relevant
- P22: [Waits monitoring](https://commitfest.postgresql.org/patch/297/) (cf, Returned with feedback, 2015-12-24) - used in the review
- P23: [Revert "Display the time when the process started waiting for the lock, in pg_locks."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=890d2182a2c425aaa80f9bf9f7116d31e0c6538e) (revert, Reverted, 2021-02-09) - judged not relevant
- P24: [Fix Windows network event handling](https://commitfest.postgresql.org/patch/3523/) (cf, Returned with feedback, 2024-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 154 s; cost reported by the CLI: $5.45
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 76 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:51:06.521Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
