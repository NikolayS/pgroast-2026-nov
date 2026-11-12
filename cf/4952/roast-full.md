# Design pre-review: Use WALReadFromBuffers in more places

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 4952](https://commitfest.postgresql.org/patch/4952/) (Needs review); authors: Bharath Rupireddy (bharathrupireddy)
- Thread: [Use WALReadFromBuffers in more places](https://www.postgresql.org/message-id/flat/CALj2ACVfF2Uj9NoFy-5m98HNtjHpuD17EDE9twVeJng-jTAe7A%40mail.gmail.com), started 2024-04-24
- Messages read: 31 of 31
- Patch prose read: v10-0001-Use-WALReadFromBuffers-for-logical-walsenders.patch, v10-0002-Use-WALReadFromBuffers-for-local-WAL-reads.patch, v10-0003-Use-WALReadFromBuffers-for-WAL-summarizer.patch, v1-0001-Use-WALReadFromBuffers-in-more-places.patch, v1-0002-Add-test-module-to-demonstrate-reading-from-WAL-b.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at level 1: the problem is stated, and several people tested or reproduced the gains. Two fixable design points remain at level 2. replication-ondisk-1 asks what ties a page read from WAL buffers to the requested timeline, and the seg.ws_tli fix is not yet posted. maintenance-fit-1 asks for one shared helper instead of four copies of the buffer-then-file logic. A committer who knows the WAL reader code can settle both.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the segment-close rule, the planned seg.ws_tli fix and the behaviour when the WAL writer reuses pages during a read could not be checked. The timeline question rests on the thread, not on the patch.

## The proposal, restated

**Problem.** Commit 91f2cae7a4 added WALReadFromBuffers() but only physical walsenders use it. Logical walsenders, read_local_xlog_page() callers and the WAL summarizer always read WAL from files, even when the WAL is still in WAL buffers. With WAL direct IO each such file read is a physical disk read, which competes with WAL writes.

**Proposed design.** Each WAL reader first tries WALReadFromBuffers() and falls back to WALRead() for the rest. After a read served fully from buffers, the reader closes the open segment when the page is on a new segment, so a later file read does not reuse a stale descriptor. The set is three patches: logical walsenders, read_local_xlog_page() callers (logical decoding SQL functions, 2PC, repack workers, pg_walinspect), and the WAL summarizer.

**User-visible surface.**
- pg_stat_io: hits on object wal now also count reads from WAL buffers by logical walsenders, local WAL readers and the WAL summarizer.
- pg_stat_io: the checkpointer now reports hits on object wal, because 2PC reads WAL there.

**What the author claims.**
- With WAL direct IO on, logical walsender file reads drop by about 99.9% and publisher TPS rises by about 10 to 17%.
- With WAL direct IO on, file reads drop by about 53% for local WAL reads and 64% for the WAL summarizer.
- With WAL direct IO off, file reads still drop and throughput does not change, so the patches do not regress.
- make check-world passes with opportunistic WAL buffer initialization disabled and 2 GiB WAL buffers, except one test that also fails on HEAD.

**Thread state.** Michael Paquier committed the pg_stat_io page-hit counting (old v10-0001); three patches remain, and Kirill Reshke reproduced the gains. Hayato Kuroda found that a read served fully from buffers leaves seg.ws_tli unset, which breaks the page_read callback rule, and the author plans a fix not yet posted.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-04-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Buffer-only reads leave seg.ws_tli unset, and the timeline invariant is not stated

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-1`

A logical slot is synced to a standby, the standby is promoted, and decoding crosses the timeline switch. Pages served from WAL buffers leave seg.ws_tli at 0 or stale, so a later file read or a page check uses the wrong timeline. The user sees a wrong error or bad decoding only after failover.

Message #30 shows that the page_read contract in xlogreader.h requires the callback to set seg.ws_tli. Message #31 agrees the contract is broken and says it only affects error reporting and validation, but no fixed version is posted. For read_local_xlog_page() callers, XLogReadDetermineTimeline() picks the timeline per page, yet the proposal does not say why a buffer read cannot return pages from a timeline other than the one the reader asked for. That gap matters now that logical decoding runs across promotion.

Thread: [#30 ("Hayato Kuroda (Fujitsu)", 2026-10-27)](https://www.postgresql.org/message-id/OS7PR01MB183178A50F430AC19E5CEDC60F5602%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#31 (Bharath Rupireddy, 2026-11-04)](https://www.postgresql.org/message-id/CALj2ACV3mL9BP0VPP%2BqA%3DU_y%3DD8vhZ_1%3DDgxr%3DYvQwvf%3DdmUAQ%40mail.gmail.com)

**Question:** With seg.ws_tli fixed, what guarantees that a buffer-served page belongs to the timeline the reader requested, including reads that straddle a promotion?

Counter-check: held. Message #31 confirms that ws_tli stays 0 after a read served fully from buffers, and no fixed version is posted. The thread does not say how a buffer read is tied to the requested timeline, so the question stands as fixable.

### 2. Buffer-then-file read logic copied into four page_read callbacks

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API, WAL / replication / standby · `maintenance-fit-1`

A later patch changes how WALRead() tracks segments or sets seg.ws_tli, and updates only some of the physical walsender, logical walsender, read_local_xlog_page_guts() and summarizer copies. A copy that was missed then reuses a stale descriptor or reports a wrong TLI. Users find out through an "unexpected pageaddr" error during decoding, or a wrong error message, once tests stop reaching the file-read fallback.

Each caller gets its own WALReadFromBuffers() plus WALRead() sequence, its own segment-close rule and its own ws_tli handling, and the author declined a shared helper (message #5, #20). The thread has already shown misses of exactly this kind: an unconditional close that Michael flagged (message #21), the dropped retry in the physical path (message #20), and the unset ws_tli (message #30, #31). Message #25 shows that tests stop exercising the file path once buffers serve every read, so a missed update would not fail loudly.

Thread: [#5 (Bharath Rupireddy, 2024-06-12)](https://www.postgresql.org/message-id/CALj2ACXa-2eEHHaNRwjcF1k9rtH%3DEJrWvbGJkucdSOD3zP-OUw%40mail.gmail.com), [#20 (Bharath Rupireddy, 2026-09-04)](https://www.postgresql.org/message-id/CALj2ACUo2Kqy8ovFwSr1Cr7hkTLGQo4U8QFDQF9nW4d9ADGfXg%40mail.gmail.com), [#21 (Michael Paquier, 2026-09-04)](https://www.postgresql.org/message-id/appZ6saaV5wV5UA4%40paquier.xyz), [#25 (Bharath Rupireddy, 2026-09-14)](https://www.postgresql.org/message-id/CALj2ACUMtx%2B3hd7mO8Gb5kewpvwQ9OLEPPSC1fC2r500k5_CaQ%40mail.gmail.com), [#30 ("Hayato Kuroda (Fujitsu)", 2026-10-27)](https://www.postgresql.org/message-id/OS7PR01MB183178A50F430AC19E5CEDC60F5602%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#31 (Bharath Rupireddy, 2026-11-04)](https://www.postgresql.org/message-id/CALj2ACV3mL9BP0VPP%2BqA%3DU_y%3DD8vhZ_1%3DDgxr%3DYvQwvf%3DdmUAQ%40mail.gmail.com)

**Question:** Can the buffer read, the segment close and the ws_tli rule live in one xlogreader-level helper, so that future WALRead() changes need one edit and fail loudly if missed?

Counter-check: held. The author's reason against a helper in message #20 is that the physical walsender cannot share it, which does not cover the three new callers. The thread shows three misses of this kind (messages #20, #21, #30), so the scenario is concrete.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Logical walsenders, read_local_xlog_page() callers and the WAL summarizer change how they read WAL, including across timeline switches. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | read_local_xlog_page() is a page_read callback that extensions such as pg_walinspect use, and its seg.ws_tli behaviour changes. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Four page_read callbacks each carry their own buffer read, segment-close rule and seg.ws_tli handling. |
| performance on the hot path | **yes** | Every WAL page read by walsenders and local readers now tries WAL buffers first, including when a lagging reader mostly misses. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. With seg.ws_tli set, what guarantees that a page read from WAL buffers belongs to the timeline the reader requested, including reads that straddle a promotion?
2. Can the buffer read, the segment close and the seg.ws_tli rule live in one helper shared by the logical walsender, read_local_xlog_page() and the WAL summarizer?
3. What does the extra WALReadFromBuffers() attempt cost when a walsender lags and almost every read misses WAL buffers, as Hayato Kuroda asked?
4. Is there a measurement with the default wal_buffers and WAL direct IO off that shows no regression?
5. Which test still reaches the WALRead() file fallback and the segment-close path once WAL buffers serve most reads?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier committed the pg_stat_io part and reviewed the rest; the owner must know xlogreader page_read callbacks and timeline handling in walsenders.
- **Liability:** The owner signs for correct WAL reads in logical decoding, 2PC, pg_walinspect and the WAL summarizer, including after promotion, for as long as these four read paths exist.
- **Author follow-up:** The author has kept the entry alive since 2024, posted ten versions, and in message #31 said a fix for seg.ws_tli is planned.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patches are small and a qualified committer is engaged, but the throughput gain is measured only with WAL direct IO, a developer setting today. The duplicated read logic and the timeline question need care in the WAL reader code, which sets the burden and review cost at medium.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent is the same idea, and no recorded reason applies. [P4] proposed tracking the oldest initialized WAL buffer page and was judged unneeded; this proposal adds no such field. [P5] concerned how WAL I/O appears in pg_stat_io, and the hit counting it touched is already committed here (message #28). Messages #12, #14 and #17 point to earlier direct IO and benchmark threads as motivation only, not as rejections.
- standards-compat: 0 objection(s). The user-facing contract is new rows in pg_stat_io: wal hits from more backend types, including the checkpointer. The hit semantics were agreed and committed in message #28. No SQL syntax, GUC, catalog, pg_dump or psql change is involved, and the change targets a major release, so this angle finds nothing to object to. Not checked: whether the pg_stat_io docs list which backend types report wal hits.
- replication-ondisk: 1 objection(s). The patches add no on-disk format, WAL record, catalog or protocol change. The only state that crosses a boundary is the reader's segment and timeline bookkeeping.
- security: 0 objection(s). The patches change only where existing WAL readers get bytes they could already read from files: logical walsenders, read_local_xlog_page() callers such as pg_walinspect, and the WAL summarizer. No reader gains access to WAL beyond its current limit. The privilege checks on pg_walinspect, logical decoding functions and replication connections stay as they are. The only new visible output is the WAL hit count in pg_stat_io, which reveals no data contents. I found no privilege, information-leak or unprivileged denial-of-service aspect to object to.
- maintenance-fit: 1 objection(s). Fit is otherwise reasonable. A committer who knows the area (Michael Paquier) is engaged and committed the stats part, the cost falls on users of the WAL readers, and the series is split into separately revertable steps.
- already-exists: 0 objection(s). No existing feature or extension meets this need. The patch adds new callers to the existing WALReadFromBuffers() and keeps the fallback to WALRead(), so it adds no second mechanism. With buffered IO the OS page cache already serves most of these reads, but the thread shows fewer syscalls with no regression. The larger gain needs debug_io_direct='wal', a question of desirability for other angles.
- demand: 0 objection(s). The material shows enough demand. Several people outside the author's company want this change: Jingtang Zhang tested it, Kirill Reshke reproduced the gains, Hayato Kuroda reached the same idea on his own, and Michael Paquier committed the pg_stat_io part. One caveat remains: the large gains are measured only with debug_io_direct = 'wal', which is a developer setting, and with buffered IO the patches only reduce syscalls, with no throughput change. Because the change is small and opportunistic and its users bear its cost, this caveat does not support a demand objection.

### Precedents retrieved

- P1: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P2: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant; reason summarised from its thread
- P3: [Non-volatile WAL buffer](https://commitfest.postgresql.org/patch/2615/) (cf, Withdrawn, 2021-06-18) - judged not relevant; reason summarised from its thread
- P4: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - used in the review; reason summarised from its thread
- P5: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - used in the review; reason summarised from its thread
- P6: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant; reason summarised from its thread
- P7: [Revert "Add test case for obsoleting slot with active walsender"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96795176810b979a2bf1f4563bdcb161679d5b95) (revert, Reverted, 2021-06-20) - judged not relevant
- P8: [Move walreceiver state assignment (to WALRCV_STREAMING) in WalReceiverMain()](https://commitfest.postgresql.org/patch/4698/) (cf, Withdrawn, 2024-01-22) - judged not relevant; reason summarised from its thread
- P9: [Track number of WAL files ready to be archived in pg_stat_archiver](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Statistics_Collector) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P10: [Change walsender so that it applies per-role settings](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Standby_server_mode) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - judged not relevant
- P13: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P14: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P15: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant
- P16: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P17: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P18: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P19: [Adding linux cachestat in file Readv and Prefetch](https://commitfest.postgresql.org/patch/4779/) (cf, Rejected, 2024-03-14) - judged not relevant
- P20: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P21: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2283/) (cf, Withdrawn, 2020-06-11) - judged not relevant
- P22: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant
- P23: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P24: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2683/) (cf, Returned with feedback, 2020-09-17) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 121 s; cost reported by the CLI: $3.94
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 54 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:28:29.557Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
