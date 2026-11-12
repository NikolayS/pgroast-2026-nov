# Design pre-review: pg_stat_io_histogram

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6433](https://commitfest.postgresql.org/patch/6433/); [thread](https://www.postgresql.org/message-id/flat/CAKZiRmwvE4uJLKTgPXeBA4m%2Bd4tTghayoefcaM9%3Dz3_S7i72GA%40mail.gmail.com), 36 of 36 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). Nothing that survived questions the goal: several reviewers report demand, and the measured CPU cost is within about 1% of master. Three fixable design points remain: standards-compat-1 (existing backends and the checkpointer record no histograms after a reload), standards-compat-2 (no agreed bucket range or view name) and replication-ondisk-1 (the pgstat file layout depends on the tracked combinations). A committer who knows the pgstat code needs to rule on these and on the v10 to v15 memory design.

Not checked: Without the code, this review could not check whether the pgstat file load validates the histogram slot count, or whether histograms could be allocated lazily outside critical sections. Only 3 of 36 messages were read directly.

## Prior attempts

None found before 2026-01-26.

## Objections a reviewer is likely to raise

### 1. Histogram collection ignores track_io_timing changes in running backends

During an incident a DBA runs ALTER SYSTEM SET track_io_timing = on and reloads to find slow fsyncs. The checkpointer, walwriter and pooled client connections keep running, so they record no histogram data, but pg_stat_io still shows their I/O time. The DBA sees an empty or partial histogram and wrongly decides fsync is not the problem.

Message #24 and message #36 state that histograms are filled only in backends started after the setting is on, and that SET in a session does not enable collection. The same GUC therefore turns on timing in pg_stat_io at once but turns on histograms only after a restart of each process. Slow COMMIT and checkpointer fsync, the cases in message #1, come from long-lived processes, so they are the ones most likely to be missed. The docs note in message #36 describes the gap but does not justify giving one existing GUC two different meanings.

**Question:** How does a DBA get histogram data for the checkpointer and existing connections without a server restart, and why should track_io_timing act differently for pg_stat_io and pg_stat_io_histogram?

<sub>fixable, level 2, new GUC; messages [#1](https://www.postgresql.org/message-id/CAKZiRmwvE4uJLKTgPXeBA4m%2Bd4tTghayoefcaM9%3Dz3_S7i72GA%40mail.gmail.com), [#24](https://www.postgresql.org/message-id/CAKZiRmz-KH2a_qcwGeFwsoajCsQEd8jXyFLx0B0yKgkK%3DLhBLA%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/CAKZiRmyuuToHXESA0pCQ1ZT6_auhAOHzV0Eb%2B5r2tRaTOtfrFw%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Bucket edges and view name become a fixed contract before agreement

Monitoring tools build alerts on rows such as bucket_latency_us = '[131072,)' or on the 8us first bucket. A later release moves the range to 64us through 1s, or to 32s, as reviewers suggested. Those alerts then match nothing or the wrong latency, and users find out only when an outage goes unreported.

The view exposes bucket edges as data values, but message #25 proposes starting at 64us, message #9 and message #29 want buckets up to tens of seconds, and the author keeps 8us to 131ms. Message #36 says the labels are approximate because buckets are computed in nanoseconds, so [8,16) really covers 8.192us to 16.384us. Message #26 also floats renaming the view to pg_stat_io_lat_histogram to leave room for size histograms. [P2] was set aside so that the meaning of pg_stat_io would not change between releases, and the same reasoning applies to these edges.

Precedent: [Show WAL stats on pg_stat_io](https://www.postgresql.org/message-id/flat/CAN55FZ3AiQ+ZMxUuXnBpd0Rrh1YhwJ5FudkHg=JU0P+-W8T4Vg@mail.gmail.com) (Returned with feedback, 2024-01-18)

**Question:** Which bucket range, exact edge labels and view name will be kept stable across releases, and how would a later change avoid breaking queries that filter on bucket_latency_us?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#9](https://www.postgresql.org/message-id/CANwKhkM1FNS1Wmc56%2BaunXhaP_zjjO2YKzKTJRoVW0RsM4Of3w%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/4ffe367d-77ac-488a-8826-64efb78d6f78%40vondra.me), [#26](https://www.postgresql.org/message-id/CAKZiRmy8qMeFprFqm0cyQQXtBRbP2_3TVNwxYOeedD-Yro-P%3DQ%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/CAKZiRmxTEAbrif9zwZ7H7%3DdmC4espPgxQYEbs7F%3DMNk%3D80sAjw%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/CAKZiRmyuuToHXESA0pCQ1ZT6_auhAOHzV0Eb%2B5r2tRaTOtfrFw%40mail.gmail.com); `standards-compat-2`</sub>

### 3. Persisted histogram layout depends on pgstat_tracks_io_* logic

A later minor release backpatches a fix that makes pgstat_tracks_io_op() accept one more combination. The slot count and offsets change, but nobody bumps PGSTAT_FILE_FORMAT_ID. After the restart, the server loads the old pgstat file and shows fsync counts under the wrong backend type or operation, and no error is raised.

Today pg_stat_io stores a fixed [object][context][op] array, so changing which combinations are tracked does not change the file layout. Per v15-0003, the histogram slot count now comes from pgstat_io_get_sum_tracked() at startup, which ties the pgstat file layout to code that looks like pure policy. Message #36 adds safeguards against reading corrupted data but does not say whether a load checks the slot count or the offset map. Message #35 also leaves the format ID bump to the committer, so the coupling is not yet recorded anywhere.

**Question:** Does loading the pgstat file reject or discard histograms when the stored slot count or offset map differs from the one the running server computes?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#31](https://www.postgresql.org/message-id/CAKZiRmx3Z%2BfxunEc4CDGrpZVFwD4MAEDfGVM_v-CW-%3DxTLeF_Q%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/CAKZiRmyyUFLiF-jp15Z40VgHeB1SMYs4rc_efZmgW1B0z0fWag%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/CAKZiRmyuuToHXESA0pCQ1ZT6_auhAOHzV0Eb%2B5r2tRaTOtfrFw%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. How does a DBA get histogram data for the checkpointer and existing connections after turning on track_io_timing with a reload, without a server restart?
2. Which bucket range will stay fixed across releases, given the open proposals to start at 64us and to extend to tens of seconds?
3. Does loading the pgstat file discard histograms when the stored slot count or offset map differs from what the running server computes?
4. Will the view be named pg_stat_io_histogram or pg_stat_io_lat_histogram, given the idea of later size histograms?
5. Will bucket_latency_us show exact edges, given that buckets are computed in nanoseconds but labelled in microseconds?

## Touch points

- **on-disk format / pg_upgrade:** Histograms persist in the pgstat file, so its format changes and a PGSTAT_FILE_FORMAT_ID bump is needed.
- **catalog changes:** The patch adds the pg_stat_io_histogram view in system_views.sql and pg_stat_get_io_histogram() in pg_proc.dat.
- **ABI / extension API:** The patch changes the I/O stats structures in pgstat.h and pgstat_internal.h and adds to pg_bitutils.h.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: Bucket edges and view name become a fixed contract before agreement
- **new GUC:** No new GUC, but track_io_timing and track_wal_io_timing gain a second effect that applies only to newly started backends.
- **psql UX conventions:** see objection standards-compat-1: Histogram collection ignores track_io_timing changes in running backends
- **backpatchability:** A backpatched change to pgstat_tracks_io_op() would change the histogram slot count and so the pgstat file layout.
- **maintenance burden ("who maintains this in 10 years"):** Bucket edges become user-visible values, and pgstat_io gets shared memory sized at startup, unlike other fixed stats kinds.
- **performance on the hot path:** Each timed I/O in pgstat_count_io_op_time() also computes a bucket and increments a counter when timing is on.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund reviewed early versions and asked to store only usable histograms, but has not replied to v10 to v15. The owner must know the cumulative statistics code and the I/O accounting in pgstat_io.c.
- Priority: needs champion; audience a sizeable minority; review cost medium. Three reviewers support the goal and the cost falls mostly on users of the timing settings, but no reviewer has answered v10 to v15 since May. The three patches total about 70 KiB and are confined to the pgstat code, with a new view and a file format change to carry.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
