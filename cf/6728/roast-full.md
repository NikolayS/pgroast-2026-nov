# Design pre-review: md: measure just FileSync() for pgstat_io without FileClose()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6728](https://commitfest.postgresql.org/patch/6728/) (Needs review); authors: Jakub Wartak (jakub.wartak.tomtom)
- Thread: [md: measure just FileSync() for pgstat_io without FileClose()](https://www.postgresql.org/message-id/flat/CAKZiRmxwAQ4W-gSJbsox57WsjVOsaZhJE5ZDN2yFo2OOfYckGQ%40mail.gmail.com), started 2026-04-30
- Messages read: 2 of 2
- Patch prose read: v1-0001-md-measure-just-FileSync-for-pgstat_io-without-Fi.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived at levels 1a, 1b or 2, so nothing ranked stops the patch. The change is confined to md.c and affects only the IOOP_FSYNC timing in pg_stat_io. Confidence is low because the thread has two messages, no measurement of close() time, and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it could not be checked whether the patch adds a new pgstat_count_io_op_time() variant or moves the FileClose() call. Whether close() time is ever large in this path also could not be judged.

## The proposal, restated

**Problem.** In mdsyncfiletag(), the time counted for IOOP_FSYNC can include the FileClose() call that follows FileSync(). The author asks whether close() time should be left out, so that fsync timing in pg_stat_io reflects only the fsync. The thread does not say who is affected or whether anyone has seen skewed numbers.

**Proposed design.** The patch changes only md.c so that the IOOP_FSYNC duration covers FileSync() alone and no longer includes a possible close() system call. The commit message gives no further design detail.

**User-visible surface.**
- Fsync timing for IOOP_FSYNC in pg_stat_io (with track_io_timing on) no longer includes close() time.

**What the author claims.**
- The current IOOP_FSYNC duration may include the duration of an extra close() system call.

**Thread state.** One reviewer, Zizhuan Liu, agrees that the timing should ideally cover only FileSync(). He doubts the change is needed, because his code reading finds no expensive cleanup in FileClose() for these handles, and he did no benchmark. The author has not replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-04-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
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
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch changes how mdsyncfiletag() in md.c times IOOP_FSYNC for pg_stat_io. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does the patch take the end time of FileSync() before FileClose() with the current pgstat_count_io_op_time() interface?
2. Is there a measurement where close() time in mdsyncfiletag() changes the IOOP_FSYNC time in pg_stat_io by a visible amount?
3. Can pgaio_closing_fd() block in FileClose() for the file handles that mdsyncfiletag() opens and closes?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the pg_stat_io timing code and the md.c sync path would have to own it; no committer has spoken in the thread.
- **Liability:** The owner signs for the meaning of the IOOP_FSYNC time in pg_stat_io, and for any new timing helper if the patch adds one.
- **Author follow-up:** The author moved the entry to the next CommitFest on 2026-10-01 but has not replied to message #2.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 1 KiB in one file and affects only users who read fsync timing in pg_stat_io with track_io_timing on. It waits on an author reply to message #2 and on a committer who knows the area.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no row that proposes the same or a similar change, and the thread does not mention an earlier discussion of the idea. Without a recorded earlier rejection, the prior-attempts angle has no objection to raise.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no syntax, GUC, catalog object or psql change, and adds nothing to dump. The only visible effect is slightly smaller IOOP_FSYNC times in pg_stat_io with track_io_timing on, and that is a fix to what the column is documented to measure. Whether the change is needed at all is a desirability question for another angle.
- replication-ondisk: 0 objection(s). The patch changes only how md.c times IOOP_FSYNC for pg_stat_io. It adds no on-disk format, WAL record, catalog, wire protocol or replication state. Cumulative stats are not replicated, and the brief names no change to the stats file format. A changed pgstat_count_io_op_time() signature could affect extensions, but the material does not show one.
- security: 0 objection(s). The patch only changes which call the IOOP_FSYNC duration in pg_stat_io covers, inside md.c. It adds no new privilege, view, column, setting or exposure, and does not change who can read pg_stat_io. So there is no security or privilege aspect to object to.
- maintenance-fit: 0 objection(s). From the maintenance angle this is a one-file change in md.c with no new GUC, catalog entry, external dependency or cost on a common path. The prose does not say whether it needs a new pgstat_count_io_op_time() variant that takes an end time; that cannot be checked without the code, and it would be the only new internal API.
- already-exists: 0 objection(s). No existing feature, view, GUC or extension gives an IOOP_FSYNC duration in pg_stat_io that leaves out close() time. The wait event WAIT_EVENT_DATA_FILE_SYNC covers FileSync() only, but it gives sampled wait state, not the cumulative pg_stat_io timing that the patch changes. The patch fixes how an existing counter is measured and adds no second mechanism, so this angle has no objection.
- demand: 0 objection(s). The thread shows no demand beyond one question, and no measurement that close() time skews IOOP_FSYNC. But the change is a small accuracy fix confined to md.c and paid for only by fsync timing, so it has no cost that falls on everyone and no concrete failure scenario. A committer would more likely ask for a short measurement, or for the reasoning about whether pgaio_closing_fd() can block in this path, than reject the patch for lack of users.

### Precedents retrieved

- P1: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant; reason summarised from its thread
- P2: [Unlogged relation copy is not fsync'd](https://commitfest.postgresql.org/patch/4517/) (cf, Returned with feedback, 2024-01-05) - judged not relevant; reason summarised from its thread
- P3: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P4: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant; reason summarised from its thread
- P5: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P6: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P7: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - judged not relevant; reason summarised from its thread
- P8: [Enhance statistics reset functions to return reset timestamp](https://commitfest.postgresql.org/patch/5956/) (cf, Withdrawn, 2025-10-28) - judged not relevant; reason summarised from its thread
- P9: [TimestampTz->Text->TimestampTz casting fails with DateStyle 'Postgres'](https://commitfest.postgresql.org/patch/5371/) (cf, Returned with feedback, 2025-01-07) - judged not relevant
- P10: [Push down time-related SQLValue functions to foreign server](https://commitfest.postgresql.org/patch/3289/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P11: [test pgbench time-related features](https://commitfest.postgresql.org/patch/3242/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P12: [Allow TIMESTAMP WITH TIME ZONE to store the original timezone information, either zone name or offset from UTC](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Dates_and_Times) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P13: [pgbench - another attempt at tap test for time-related options](https://commitfest.postgresql.org/patch/1306/) (cf, Returned with feedback, 2020-07-13) - judged not relevant
- P14: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P15: [Add last_executed timestamp to pg_stat_statements](https://commitfest.postgresql.org/patch/6308/) (cf, Withdrawn, 2026-03-30) - judged not relevant
- P16: [Replace relation_{open,close} to table_{open,close} in policy.c](https://commitfest.postgresql.org/patch/6072/) (cf, Rejected, 2026-03-27) - judged not relevant
- P17: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P18: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P19: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/2496/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P20: [pgbench - option to build using ppoll() for larger connection counts](https://commitfest.postgresql.org/patch/1307/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P21: [Revert "Disallow infinite endpoints in generate_series() for timestamps."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29904f5f2fdafbbb96ef3685fd361053b061aeb1) (revert, Reverted, 2022-05-09) - judged not relevant
- P22: [Warn if initdb's --sync-only option is mixed with other options](https://commitfest.postgresql.org/patch/3239/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P23: [Issue in postgres_fdw causing unnecessary wait for cancel request reply](https://commitfest.postgresql.org/patch/4329/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P24: [Time-delayed logical replication subscriber](https://commitfest.postgresql.org/patch/3581/) (cf, Returned with feedback, 2023-03-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 70 s; cost reported by the CLI: $0.58
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 38 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:01:33.724Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
