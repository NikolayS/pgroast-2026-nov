# Design pre-reviews of CommitFest entries

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

Each entry was read from its mailing-list thread and the prose of its latest patch (commit message, file list, documentation changes). No code was read. A report lists objections that a reviewer may raise, with links to the thread. It is one-sided on purpose and it is not a decision about the patch. The verdict line is the least reliable part: read the objections.

Made with [pgroast](https://github.com/NikolayS/pgroast). In each folder: `roast.md` (short report), `roast-full.md` (everything, with the models used), `roast.json`, `roast-summary.txt`.

Entries done: 425.

- NEEDS-DESIGN-DISCUSSION: 193
- NEEDS-COMMITTER-EYES: 151
- NO-FUNDAMENTAL-OBJECTION: 79
- LIKELY-REJECT: 2

| Entry | Title | Verdict | Objections | First objection | Prior attempts | Messages read | Date |
|---|---|---|---:|---|---:|---:|---|
| [4351](cf/4351/roast.md) | Index Prefetching | NEEDS-COMMITTER-EYES | 4 | GiST batches on a hot standby can return deleted tuples | 0 | 515 | 2026-11-12 |
| [4460](cf/4460/roast.md) | Implement row pattern recognition feature | NEEDS-COMMITTER-EYES | 5 | Reduced-frame design is still open and would change the window API | 0 | 549 | 2026-11-12 |
| [4573](cf/4573/roast.md) | pg_ctl start may return 0 even if the postmaster has been already started on Windows | NEEDS-COMMITTER-EYES | 4 | Dropping cmd.exe brings back the AutoRun change reverted for v17 | 0 | 43 | 2026-11-12 |
| [4681](cf/4681/roast.md) | Make COPY format extendable: Extract COPY TO format implementations | NEEDS-COMMITTER-EYES | 5 | Every new COPY option needs a decision in every custom format | 1 | 341 | 2026-11-12 |
| [4817](cf/4817/roast.md) | COPY (on_error table) saving error info to a table | NEEDS-DESIGN-DISCUSSION | 5 | Fixed pg_catalog row type makes later column changes break upgrades | 2 | 31 | 2026-11-12 |
| [4952](cf/4952/roast.md) | Use WALReadFromBuffers in more places | NEEDS-COMMITTER-EYES | 2 | Buffer-only reads leave seg.ws_tli unset, and the timeline invariant is not stated | 0 | 31 | 2026-11-12 |
| [4971](cf/4971/roast.md) | [CREATE\|RE] INDEX CONCURRENTLY with single heap scan | NEEDS-DESIGN-DISCUSSION | 5 | Validation snapshot refresh needs a stated argument against the PG14 failure | 3 | 126 | 2026-11-12 |
| [5012](cf/5012/roast.md) | Vacuum statistics | NEEDS-DESIGN-DISCUSSION | 3 | No vacuum setting is named that the extension statistics would tune | 1 | 161 | 2026-11-12 |
| [5018](cf/5018/roast.md) | Extension security improvement: Add support for extensions with an owned schema | NEEDS-DESIGN-DISCUSSION | 5 | Owned schema shared via CASCADE breaks restore and pg_upgrade drops a dependency | 0 | 43 | 2026-11-12 |
| [5029](cf/5029/roast.md) | Historic snapshot doesn't track txns committed in BUILDING_SNAPSHOT state | NEEDS-COMMITTER-EYES | 1 | Hand-kept list of catalog-marking WAL records in the BUILDING_SNAPSHOT path | 0 | 10 | 2026-11-12 |
| [5129](cf/5129/roast.md) | Asynchronous MergeAppend Execution | NEEDS-COMMITTER-EYES | 2 | Async FDWs now receive MergeAppendState with no opt-in or loud failure | 0 | 64 | 2026-11-12 |
| [5151](cf/5151/roast.md) | DirtySnapshot index scan skips concurrently updated tuples leading to lost updates/deletes in logical replication | NEEDS-COMMITTER-EYES | 2 | MVCC lookup drops the wait for an in-flight local INSERT | 0 | 45 | 2026-11-12 |
| [5276](cf/5276/roast.md) | Fix race between WAL flush and InstallXLogFileSegment() | NEEDS-COMMITTER-EYES | 2 | Skip-the-lock value tracks segment number only, but names carry timeline | 0 | 6 | 2026-11-12 |
| [5279](cf/5279/roast.md) | Add XMLSerialize: version (SQL/XML X076), explicit declaration (SQL/XML X078) | NEEDS-DESIGN-DISCUSSION | 5 | Most of the result is reachable today with xmlroot() and string functions | 0 | 14 | 2026-11-12 |
| [5330](cf/5330/roast.md) | Logging plan of the currently running query | NEEDS-COMMITTER-EYES | 4 | No overhead numbers for the per-query bookkeeping every backend pays | 2 | 197 | 2026-11-12 |
| [5444](cf/5444/roast.md) | using index to speedup add not null constraints to a table | NEEDS-DESIGN-DISCUSSION | 3 | Correctness rests on an implicit list of phase 3 conditions | 0 | 22 | 2026-11-12 |
| [5487](cf/5487/roast.md) | Pull-up subquery if INNER JOIN-ON contains refs to upper-query | NEEDS-DESIGN-DISCUSSION | 2 | Moving the correlated qual to WHERE already gives the semi join | 2 | 30 | 2026-11-12 |
| [5526](cf/5526/roast.md) | Add storage I/O tracking to EXPLAIN IO option | NEEDS-DESIGN-DISCUSSION | 3 | Counters vanish under the default io_method, where most users run | 0 | 50 | 2026-11-12 |
| [5556](cf/5556/roast.md) | Expanding HOT updates for expression and partial indexes | NEEDS-DESIGN-DISCUSSION | 3 | Users of costly expression indexes pay with no opt-out shown | 0 | 88 | 2026-11-12 |
| [5570](cf/5570/roast.md) | Bump soft open file limit (RLIMIT_NOFILE) to hard limit on startup | NEEDS-COMMITTER-EYES | 2 | Default Linux installs raise the limit for children outside pg_system/pg_popen | 0 | 35 | 2026-11-12 |
| [5597](cf/5597/roast.md) | Warn, and disallow, clear text passwords via SQL | NEEDS-DESIGN-DISCUSSION | 3 | Default warn puts a lasting cost on every user with no removal plan | 0 | 33 | 2026-11-12 |
| [5621](cf/5621/roast.md) | Compress big WAL records | NEEDS-DESIGN-DISCUSSION | 5 | Existing wal_compression users get whole-record compression by default | 0 | 37 | 2026-11-12 |
| [5631](cf/5631/roast.md) | Allow table AMs to define their own reloptions | NEEDS-COMMITTER-EYES | 5 | Thread does not explain how v14 avoids the 2024 revert reasons | 4 | 39 | 2026-11-12 |
| [5664](cf/5664/roast.md) | Fix slot synchronization with two_phase decoding enabled | NEEDS-DESIGN-DISCUSSION | 1 | Minor release makes dumps from older 17.x releases fail to restore | 1 | 100 | 2026-11-12 |
| [5667](cf/5667/roast.md) | support create index on virtual generated column. | NEEDS-DESIGN-DISCUSSION | 2 | Expression indexes on the generation expression already meet the stated need | 0 | 28 | 2026-11-12 |
| [5671](cf/5671/roast.md) | Produce LLVM bitcode files on meson builds | NEEDS-DESIGN-DISCUSSION | 5 | General extension .pc file has no demonstrated users yet | 0 | 32 | 2026-11-12 |
| [5774](cf/5774/roast.md) | Extend skipping FK checks on replicas to include ADD FK and TRUNCATE | NEEDS-DESIGN-DISCUSSION | 3 | Earlier objection to extending replica-role FK bypass to truncate is unanswered | 1 | 12 | 2026-11-12 |
| [5845](cf/5845/roast.md) | Carefully exposing information without authentication | LIKELY-REJECT | 3 | A small separate service already answers this without core changes | 0 | 18 | 2026-11-12 |
| [5877](cf/5877/roast.md) | Report bytes and transactions actually sent downtream | NEEDS-COMMITTER-EYES | 2 | One output_bytes column counts different framing per consumer path | 0 | 123 | 2026-11-12 |
| [5879](cf/5879/roast.md) | amcheck: support for GiST | NEEDS-COMMITTER-EYES | 1 | Lock strength and page-deletion concerns from the 2019 attempt are unanswered | 2 | 17 | 2026-11-12 |
| [5893](cf/5893/roast.md) | Adding pg_dump flag for parallel export to pipes | NEEDS-COMMITTER-EYES | 4 | pg_restore --pipe passes archive-supplied file names to the shell | 0 | 31 | 2026-11-12 |
| [5941](cf/5941/roast.md) | CAST(... ON DEFAULT) - WIP build on top of Error-Safe User Functions | NEEDS-DESIGN-DISCUSSION | 4 | Error-catching rules depend on constant folding, plan shape and nesting | 0 | 101 | 2026-11-12 |
| [6023](cf/6023/roast.md) | Stack-based tracking of per-node WAL/buffer usage | NEEDS-COMMITTER-EYES | 4 | Extension callers of the plain Instrumentation API can leave a dangling stack | 2 | 49 | 2026-11-12 |
| [6024](cf/6024/roast.md) | Improve read_local_xlog_page_guts by replacing polling with latch-based waiting | NEEDS-COMMITTER-EYES | 1 | Unbounded flush wait depends on every flush path calling the wakeup | 1 | 22 | 2026-11-12 |
| [6029](cf/6029/roast.md) | NOT NULL NOT ENFORCED | NEEDS-DESIGN-DISCUSSION | 5 | F492 marked complete while ALTER CONSTRAINT ENFORCED is rejected for not-null | 1 | 18 | 2026-11-12 |
| [6031](cf/6031/roast.md) | UPDATE with invalid domain constraint | NEEDS-DESIGN-DISCUSSION | 4 | Backporting turns working updates into errors after a minor upgrade | 0 | 3 | 2026-11-12 |
| [6038](cf/6038/roast.md) | Timeline switching with partial WAL records can break replica recovery | NEEDS-DESIGN-DISCUSSION | 5 | The stuck replica may never reach the record that carries overwritten_lsn | 1 | 10 | 2026-11-12 |
| [6040](cf/6040/roast.md) | Fix read/write races with pg_control file in case when postgres was compiled with DEXEC_BACKEND option | NEEDS-DESIGN-DISCUSSION | 1 | Overlap with the fix in the thread linked in message #5 is unresolved | 1 | 6 | 2026-11-12 |
| [6045](cf/6045/roast.md) | Python/pytest test framework | NEEDS-DESIGN-DISCUSSION | 4 | No stated plan for buildfarm client support of pytest suites | 3 | 116 | 2026-11-12 |
| [6047](cf/6047/roast.md) | Fix replica identity mismatch for partitioned tables with publish_via_partition_root | NEEDS-DESIGN-DISCUSSION | 5 | A 'K' tuple conflicts with the root Relation message's FULL identity | 2 | 11 | 2026-11-12 |
| [6049](cf/6049/roast.md) | Adding basic NUMA awareness | NEEDS-DESIGN-DISCUSSION | 5 | Replacement-policy rewrite is bundled under the NUMA partitioning patches | 1 | 111 | 2026-11-12 |
| [6055](cf/6055/roast.md) | Whole-row fixes for DROP COLUMN, SET COLUMN DATA TYPE | NEEDS-DESIGN-DISCUSSION | 5 | DROP COLUMN drops whole-row indexes that are still valid | 2 | 17 | 2026-11-12 |
| [6056](cf/6056/roast.md) | meson: Fix libpq search order | NEEDS-COMMITTER-EYES | 3 | Order convention in about 30 meson.build files has only partial CI coverage | 2 | 16 | 2026-11-12 |
| [6061](cf/6061/roast.md) | MPTCP - multiplexing many TCP connections through one socket to get better bandwidth | NEEDS-DESIGN-DISCUSSION | 2 | mptcpize already gives MPTCP to unmodified Postgres and libpq clients | 0 | 7 | 2026-11-12 |
| [6088](cf/6088/roast.md) | create table like including storage parameter | NEEDS-DESIGN-DISCUSSION | 3 | Including all now fails on commands that succeed today | 1 | 17 | 2026-11-12 |
| [6119](cf/6119/roast.md) | Migrate pg_commit_ts directory to the new node during an upgrade | NEEDS-DESIGN-DISCUSSION | 5 | Copied commit timestamps carry roident values the new cluster does not preserve | 0 | 60 | 2026-11-12 |
| [6128](cf/6128/roast.md) | GIN pageinspect support for entry tree and non-leaf posting tree | NEEDS-COMMITTER-EYES | 2 | Output columns encode on-disk details that later fixes cannot change | 2 | 46 | 2026-11-12 |
| [6132](cf/6132/roast.md) | Issue with synced logical replication slot during switchover | NEEDS-DESIGN-DISCUSSION | 1 | Two incompatible designs with no stated choice | 0 | 66 | 2026-11-12 |
| [6137](cf/6137/roast.md) | postgres_fdw: Use COPY to speed up batch inserts | NEEDS-DESIGN-DISCUSSION | 1 | Only independent benchmark shows v16 slower than HEAD for COPY FROM | 0 | 37 | 2026-11-12 |
| [6155](cf/6155/roast.md) | Add a warning on stats manipulation functions | NEEDS-DESIGN-DISCUSSION | 1 | Warning sits on the functions page, not where pg_dump users look | 0 | 13 | 2026-11-12 |
| [6156](cf/6156/roast.md) | Add read-only disconnect_requested GUC for graceful client disconnect (reviously goaway) | NEEDS-DESIGN-DISCUSSION | 3 | Reported GUC drops the opt-in that kept old poolers from forwarding | 3 | 22 | 2026-11-12 |
| [6177](cf/6177/roast.md) | Add archive_mode to prevent unarchived WAL on permanent primary failure | NEEDS-DESIGN-DISCUSSION | 5 | Shared mode still needs a duplicate-tolerant archive_command, the reason it was dropped | 1 | 42 | 2026-11-12 |
| [6178](cf/6178/roast.md) | COPY FROM with RLS | NEEDS-COMMITTER-EYES | 4 | COPY depends on a planned but never executed dummy INSERT | 0 | 13 | 2026-11-12 |
| [6188](cf/6188/roast.md) | Report oldest xmin source when autovacuum cannot remove tuples | NEEDS-COMMITTER-EYES | 2 | Second copy of horizon rules can drift from ComputeXidHorizons() | 0 | 55 | 2026-11-12 |
| [6201](cf/6201/roast.md) | Fix orphaned backend processes on Windows using Job Objects | NEEDS-DESIGN-DISCUSSION | 3 | Stated problem and the reproduction in the commit message do not match | 1 | 10 | 2026-11-12 |
| [6204](cf/6204/roast.md) | Windows relation extension failure at 2GB and 4GB | NEEDS-COMMITTER-EYES | 3 | v6 removes the meson segment size check before all relation-file paths are converted | 2 | 32 | 2026-11-12 |
| [6207](cf/6207/roast.md) | Fix socket and shared memory handle inheritance on Windows | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 10 | 2026-11-12 |
| [6219](cf/6219/roast.md) | pg_dump flag for dumping tables data in multiple chunks, for huge tables can often give near-linear speedup when parallelized | NEEDS-COMMITTER-EYES | 4 | Restore of several data entries into one table breaks the truncate-before-load invariant | 0 | 46 | 2026-11-12 |
| [6233](cf/6233/roast.md) | Bypassing cursors in postgres_fdw to enable parallel plans | NEEDS-DESIGN-DISCUSSION | 5 | Drain runs type input functions under another statement's security context | 1 | 58 | 2026-11-12 |
| [6242](cf/6242/roast.md) | parallel pgbench -i | NEEDS-DESIGN-DISCUSSION | 3 | No comparison with existing server-side generation (-I G) | 0 | 20 | 2026-11-12 |
| [6270](cf/6270/roast.md) | skip check constraints verification for unaffected columns during UPDATE | NEEDS-DESIGN-DISCUSSION | 1 | No measurement shows the skip saves meaningful time | 0 | 12 | 2026-11-12 |
| [6272](cf/6272/roast.md) | PoC: Simplify recovery after dropping a table by LOGGING the restore LSN | NEEDS-DESIGN-DISCUSSION | 2 | Event triggers with recovery_target_xid already give an exact recovery target | 1 | 22 | 2026-11-12 |
| [6292](cf/6292/roast.md) | Proposal to allow setting cursor options on Portals | NEEDS-DESIGN-DISCUSSION | 5 | Negotiating _pq_.cursor changes every Bind and Execute on the connection | 1 | 54 | 2026-11-12 |
| [6304](cf/6304/roast.md) | Incorrect historic snapshot may be serialized to disk during fast-forwarding | NEEDS-COMMITTER-EYES | 1 | Wrong snapshot files already on disk keep being reused after the fix | 0 | 10 | 2026-11-12 |
| [6307](cf/6307/roast.md) | Expose checkpoint timestamp and duration in pg_stat_checkpointer | NEEDS-DESIGN-DISCUSSION | 3 | No user demand shown for last-only checkpoint gauges | 2 | 15 | 2026-11-12 |
| [6314](cf/6314/roast.md) | Don't use the deprecated and insecure PQcancel in our frontend tools anymore | NEEDS-COMMITTER-EYES | 1 | pselect() was reverted before because some platforms ship a broken one | 2 | 32 | 2026-11-12 |
| [6335](cf/6335/roast.md) | File locks for data directory lockfile in the context of Linux namespaces | NEEDS-DESIGN-DISCUSSION | 2 | Stable target changes startup interlock behaviour in a minor release | 0 | 10 | 2026-11-12 |
| [6341](cf/6341/roast.md) | Cleaning up PREPARE query strings? | NEEDS-DESIGN-DISCUSSION | 2 | No measured memory saving and no users beyond one workload | 1 | 19 | 2026-11-12 |
| [6363](cf/6363/roast.md) | Add a berief general comment on BTScanInsertData's nextkey and backward | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 4 | 2026-11-12 |
| [6366](cf/6366/roast.md) | QoL improvements for pgindent and combining pgindent, pgperltidy, pgperlcritic and pgperlsyncheck into one tool | NEEDS-DESIGN-DISCUSSION | 5 | get_perltidy commits the project to owning a downloader for CPAN code | 0 | 21 | 2026-11-12 |
| [6384](cf/6384/roast.md) | llvmjit: always add the simplifycfg pass | NEEDS-COMMITTER-EYES | 1 | Compile cost on wide deform functions is not measured | 1 | 10 | 2026-11-12 |
| [6397](cf/6397/roast.md) | extended statistic (examine_variable) ignored CollateExpr | NEEDS-DESIGN-DISCUSSION | 4 | Fallback match may reuse statistics built under another collation | 0 | 10 | 2026-11-12 |
| [6401](cf/6401/roast.md) | Resetting snapshots during the first phase of [CREATE \|RE]INDEX CONCURRENTLY | NEEDS-DESIGN-DISCUSSION | 5 | On by default for every concurrent build, with no measurements | 2 | 13 | 2026-11-12 |
| [6433](cf/6433/roast.md) | pg_stat_io_histogram | NEEDS-COMMITTER-EYES | 3 | Histogram collection ignores track_io_timing changes in running backends | 0 | 36 | 2026-11-12 |
| [6436](cf/6436/roast.md) | Add `tsmatch` JSONPath operator for granular Full Text Search | NEEDS-DESIGN-DISCUSSION | 5 | Default tsconfig makes immutable jsonpath functions depend on a GUC | 2 | 8 | 2026-11-12 |
| [6440](cf/6440/roast.md) | tablecmds: fix bug where index rebuild loses replica identity on partitions | NEEDS-DESIGN-DISCUSSION | 4 | Two competing mechanisms for the same rebuild path | 0 | 45 | 2026-11-12 |
| [6442](cf/6442/roast.md) | tablecmds: fix Deadlock caused by Attach Partition | NEEDS-DESIGN-DISCUSSION | 3 | Index ATTACH now takes AccessExclusiveLock on the root table every time | 0 | 16 | 2026-11-12 |
| [6444](cf/6444/roast.md) | Set 1s WaitLatch timeout if standby limit has expired in ResolveRecoveryConflictWithBufferPin | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 12 | 2026-11-12 |
| [6445](cf/6445/roast.md) | Set 1s WaitLatch timeout if standby limit has expired in ResolveRecoveryConflictWithBufferPin | NEEDS-COMMITTER-EYES | 1 | Fix and the pending loop rewrite of the same wait can drift apart | 1 | 12 | 2026-11-12 |
| [6460](cf/6460/roast.md) | Show expression of virtual columns in error messages | NEEDS-DESIGN-DISCUSSION | 1 | Unconditional write-time computation duplicates what stored columns and checks offer | 0 | 18 | 2026-11-12 |
| [6479](cf/6479/roast.md) | SLOPE - Planner optimizations on monotonic expressions. | NEEDS-DESIGN-DISCUSSION | 3 | Demand is shown for time-bucket grouping only, but the scope is much wider | 0 | 44 | 2026-11-12 |
| [6482](cf/6482/roast.md) | Add table COMMENTS in CREATE TABLE LIKE | NEEDS-DESIGN-DISCUSSION | 2 | No field request or workflow shows anyone needs the table comment copied | 2 | 37 | 2026-11-12 |
| [6524](cf/6524/roast.md) | [PATCH] Add parallel instrumentation hooks for CustomScan providers | NEEDS-COMMITTER-EYES | 3 | Provider-chosen shm_toc keys can collide with core keys in the DSM | 1 | 13 | 2026-11-12 |
| [6525](cf/6525/roast.md) | Fix XLogFileReadAnyTLI silently applying divergent WAL from wrong timeline | NEEDS-COMMITTER-EYES | 1 | Back-patched stall can leave existing standbys waiting with no visible log message | 0 | 12 | 2026-11-12 |
| [6526](cf/6526/roast.md) | amcheck: add index-all-keys-match verification for B-Tree | NEEDS-DESIGN-DISCUSSION | 4 | heapallindexed may already report the key mismatches this targets | 1 | 14 | 2026-11-12 |
| [6557](cf/6557/roast.md) | POC: PLpgSQL FOREACH IN JSON ARRAY | NEEDS-DESIGN-DISCUSSION | 5 | Iterator API in SubscriptRoutines is still shaped for PL/pgSQL | 1 | 28 | 2026-11-12 |
| [6558](cf/6558/roast.md) | Pgbench: remove synchronous prepare | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 15 | 2026-11-12 |
| [6564](cf/6564/roast.md) | Reject ADD CONSTRAINT NOT NULL if name mismatches existing domain not-null constraint | NEEDS-DESIGN-DISCUSSION | 2 | New ERROR is proposed for stable branches as a bugfix | 0 | 4 | 2026-11-12 |
| [6579](cf/6579/roast.md) | Fix the wrong behavior of REFRESH MATERIALIZED VIEW CONCURRENTLY | NEEDS-COMMITTER-EYES | 2 | Fix leaves the documented invariant of refresh_by_match_merge() wrong | 0 | 22 | 2026-11-12 |
| [6580](cf/6580/roast.md) | Fix performance of REFRESH MATERIALIZED VIEW CONCURRENTLY | NEEDS-DESIGN-DISCUSSION | 3 | The only two designs tried are either incorrect or a no-op | 0 | 22 | 2026-11-12 |
| [6598](cf/6598/roast.md) | Add ssl_(supported\|shared)_groups to sslinfo | NEEDS-DESIGN-DISCUSSION | 4 | Row-type layout fixed before the wider handshake-info design is settled | 2 | 22 | 2026-11-12 |
| [6607](cf/6607/roast.md) | New pg_stat_tablespace view for tablespace level metrics | NEEDS-DESIGN-DISCUSSION | 5 | New code in the buffer manager I/O path with no overhead numbers | 0 | 26 | 2026-11-12 |
| [6617](cf/6617/roast.md) | Fix a file descriptor leak when using io_method=io_uring | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 22 | 2026-11-12 |
| [6619](cf/6619/roast.md) | pg_stat_xact_user_tables stat not currect during the transaction | NEEDS-DESIGN-DISCUSSION | 5 | Both designs charge every backend for a rarely used view | 0 | 8 | 2026-11-12 |
| [6627](cf/6627/roast.md) | Convert ALL SubLinks to ANY SubLinks | NEEDS-DESIGN-DISCUSSION | 3 | The main case, <> ALL, already has a hashed path via NOT IN | 3 | 8 | 2026-11-12 |
| [6635](cf/6635/roast.md) | Add support for INSERT ... SET syntax | NEEDS-DESIGN-DISCUSSION | 3 | Multi-row SET form may diverge from a future SQL standard | 3 | 23 | 2026-11-12 |
| [6639](cf/6639/roast.md) | Track skipped vacuum and analyze in pg_stat_all_tables | NEEDS-DESIGN-DISCUSSION | 4 | Roles without privileges on a table can bump its lock-skip statistics | 2 | 51 | 2026-11-12 |
| [6642](cf/6642/roast.md) | Warn when creating or enabling a subscription with max_logical_replication_workers = 0 | NEEDS-COMMITTER-EYES | 1 | Warning condition copied into command sites, so it drifts from launcher.c | 0 | 13 | 2026-11-12 |
| [6644](cf/6644/roast.md) | Extract numeric filed in JSONB more effectively | NEEDS-COMMITTER-EYES | 3 | Rewritten casts must behave exactly like the casts they replace | 1 | 114 | 2026-11-12 |
| [6648](cf/6648/roast.md) | Docs: Create table description for constraints markup fix and label tweaks | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 3 | 2026-11-12 |
| [6652](cf/6652/roast.md) | Resizable shared memory structures | NEEDS-COMMITTER-EYES | 5 | MAP_NORESERVE removes the huge page reservation check at startup | 2 | 93 | 2026-11-12 |
| [6664](cf/6664/roast.md) | Support EXCEPT for ALL SEQUENCES publications | NEEDS-DESIGN-DISCUSSION | 3 | Use case for excluding sequences is stated but not shown | 1 | 82 | 2026-11-12 |
| [6667](cf/6667/roast.md) | DISTINCT in plain aggregate window functions | NEEDS-DESIGN-DISCUSSION | 4 | Grow-only hash path ignores work_mem with no plan to bound it | 3 | 4 | 2026-11-12 |
| [6668](cf/6668/roast.md) | Implement missing join selectivity estimation for range types | NEEDS-DESIGN-DISCUSSION | 2 | No user report shows range or multirange joins being misestimated | 0 | 32 | 2026-11-12 |
| [6674](cf/6674/roast.md) | Adding a stored generated column without long-lived locks | NEEDS-DESIGN-DISCUSSION | 5 | Clearing attmissingval can turn unrewritten tuples to NULL | 0 | 42 | 2026-11-12 |
| [6681](cf/6681/roast.md) | docs: Include database collation check on SQL from alter_collation.sgml | NEEDS-COMMITTER-EYES | 3 | v2 stops reporting ICU collation mismatches the current query catches | 1 | 3 | 2026-11-12 |
| [6689](cf/6689/roast.md) | Extensible sync handler registration | NEEDS-DESIGN-DISCUSSION | 3 | The fsync validation use case is not served by this design | 3 | 3 | 2026-11-12 |
| [6696](cf/6696/roast.md) | Proposal: Conflict log history table for Logical Replication | NEEDS-COMMITTER-EYES | 5 | Apply worker errors on conflict log table lock contention under two_phase | 1 | 747 | 2026-11-12 |
| [6712](cf/6712/roast.md) | [Patch]Incorrect usage of PageGetFreeSpace instead of PageGetHeapFreeSpace in heap_xlog_visible | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [6728](cf/6728/roast.md) | md: measure just FileSync() for pgstat_io without FileClose() | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 2 | 2026-11-12 |
| [6730](cf/6730/roast.md) | Skip prefetch for block references that follow a FPW or WILL_INIT of the same block | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 17 | 2026-11-12 |
| [6743](cf/6743/roast.md) | Preserve replication origin OIDs in pg_upgrade | NEEDS-DESIGN-DISCUSSION | 3 | Main benefit depends on a pg_commit_ts migration not yet accepted | 1 | 90 | 2026-11-12 |
| [6749](cf/6749/roast.md) | Disallow whole-row index references with virtual generated columns? | NEEDS-COMMITTER-EYES | 2 | Adding a virtual generated column later still creates the broken index | 0 | 8 | 2026-11-12 |
| [6752](cf/6752/roast.md) | Mark the enum comparison functions as leakproof | NEEDS-DESIGN-DISCUSSION | 2 | Demand covers enum equality only, not enum_cmp or ordering operators | 1 | 5 | 2026-11-12 |
| [6757](cf/6757/roast.md) | Avoid unnecessary StringInfo allocation in tablesync COPY buffer | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 5 | 2026-11-12 |
| [6761](cf/6761/roast.md) | Remove invalid SS2/SS3 handling from EUC-KR routines | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 10 | 2026-11-12 |
| [6763](cf/6763/roast.md) | Improve pg_stat_statements scalability | NEEDS-COMMITTER-EYES | 5 | Removing pg_stat_statements could discard all cumulative statistics at restart | 5 | 29 | 2026-11-12 |
| [6765](cf/6765/roast.md) | teach vacuumlo to handle domains | NEEDS-COMMITTER-EYES | 2 | Domain fix and composite/array expansion are bundled in one patch | 0 | 7 | 2026-11-12 |
| [6768](cf/6768/roast.md) | pg_rewind does not rewind diverging timelines | NEEDS-DESIGN-DISCUSSION | 4 | Archived WAL segments with the same TLI stay ambiguous to pg_rewind | 1 | 36 | 2026-11-12 |
| [6770](cf/6770/roast.md) | [BUG] DELETE/UPDATE more than one row in partitioned foreign table | NEEDS-COMMITTER-EYES | 4 | Out-of-range pseudo attribute number adds a silent planner invariant | 0 | 37 | 2026-11-12 |
| [6781](cf/6781/roast.md) | pgstat: Flush some statistics within running transactions, take 2 | NEEDS-COMMITTER-EYES | 2 | Custom stats callbacks can silently double count after the API change | 1 | 44 | 2026-11-12 |
| [6788](cf/6788/roast.md) | COPY TO BLACKHOLE / pg_dump -j -Fb | LIKELY-REJECT | 2 | A COPY destination for corruption checks does not belong in COPY | 0 | 6 | 2026-11-12 |
| [6792](cf/6792/roast.md) | to_date()/to_timestamp() silently accept month=0 and day=0 | NEEDS-DESIGN-DISCUSSION | 2 | A narrow 00 check repeats the margin-fix pattern rejected for year 0 | 2 | 18 | 2026-11-12 |
| [6804](cf/6804/roast.md) | log_postmaster_stats | NEEDS-DESIGN-DISCUSSION | 5 | CPU-based trigger kept against reviewer advice and cannot be tested | 0 | 9 | 2026-11-12 |
| [6821](cf/6821/roast.md) | Unify parallel worker handling for index builds | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 7 | 2026-11-12 |
| [6824](cf/6824/roast.md) | Fix: VACUUM FULL or CREATE INDEX fails with error "missing chunk number 0 for toast value XXX" | NEEDS-COMMITTER-EYES | 2 | Skipping tuples with missing TOAST data hides real corruption from CREATE INDEX | 0 | 7 | 2026-11-12 |
| [6837](cf/6837/roast.md) | Add pg_get_table_ddl() to reconstruct CREATE TABLE statements | NEEDS-DESIGN-DISCUSSION | 5 | Extends a function family that was reverted pending a redesign | 3 | 73 | 2026-11-12 |
| [6850](cf/6850/roast.md) | Don't consider FATAL errors "abnormal server termination" | NEEDS-DESIGN-DISCUSSION | 2 | Clients that match the appended string lose reconnect detection | 1 | 7 | 2026-11-12 |
| [6861](cf/6861/roast.md) | Separate catalog_xmin from xmin in walsender hot standby feedback | NEEDS-DESIGN-DISCUSSION | 3 | wal_receiver_create_temp_slot already gives slotless standbys a separate catalog_xmin | 0 | 4 | 2026-11-12 |
| [6864](cf/6864/roast.md) | Add explicit warnings about unsafe OAuth trace output for libpq | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 4 | 2026-11-12 |
| [6870](cf/6870/roast.md) | LLVM JIT: any JIT-compiled query crashes (SIGILL) on a libLLVM 19 + ASAN build | NEEDS-COMMITTER-EYES | 1 | Per-sanitizer source pragmas may not scale to other instrumentation | 0 | 11 | 2026-11-12 |
| [6888](cf/6888/roast.md) | Allow wal_log_hints to be changed without restart | NEEDS-DESIGN-DISCUSSION | 4 | pg_rewind accepts a target that had wal_log_hints off after divergence | 0 | 2 | 2026-11-12 |
| [6892](cf/6892/roast.md) | Randomize B-Tree page split location to avoid oscillating patterns | NEEDS-DESIGN-DISCUSSION | 2 | No field report shows users hurt by split oscillation | 0 | 14 | 2026-11-12 |
| [6897](cf/6897/roast.md) | Require SSL connection to postgres for oauth | NEEDS-DESIGN-DISCUSSION | 4 | 0002 breaks working plaintext OAuth setups, and the only opt-out is a debug flag | 0 | 9 | 2026-11-12 |
| [6903](cf/6903/roast.md) | Two bugs around ALTER TYPE | NEEDS-COMMITTER-EYES | 3 | New lock changes ALTER TYPE blocking for existing users, mainly if backpatched | 1 | 5 | 2026-11-12 |
| [6912](cf/6912/roast.md) | SQL/JSON: JSON_TRANSFORM (SQL standard, subclause 6.44) | NEEDS-COMMITTER-EYES | 4 | Claimed portability rests on a subset of T883 nobody checked against the text | 1 | 8 | 2026-11-12 |
| [6915](cf/6915/roast.md) | pg_dump: skip pg_init_privs entries for non-existent roles | NEEDS-COMMITTER-EYES | 2 | Dropping whole aclitems leaves upgraded extensions with permanent dump drift | 1 | 35 | 2026-11-12 |
| [6916](cf/6916/roast.md) | PROXY protocol support | NEEDS-DESIGN-DISCUSSION | 5 | Trusting Unix sockets lets any local OS user claim any client IP | 0 | 53 | 2026-11-12 |
| [6918](cf/6918/roast.md) | Global temporary tables | NEEDS-DESIGN-DISCUSSION | 5 | Idle sessions that used a global temporary table can block wraparound protection | 3 | 47 | 2026-11-12 |
| [6921](cf/6921/roast.md) | Continuous re-validation of session credentials | NEEDS-DESIGN-DISCUSSION | 5 | Baseline check applies rolvaliduntil to sessions that never used a password | 0 | 8 | 2026-11-12 |
| [6925](cf/6925/roast.md) | Add PRODUCT() aggregate function | NEEDS-COMMITTER-EYES | 1 | product(real) returns double precision while sum(real) returns real | 0 | 29 | 2026-11-12 |
| [6931](cf/6931/roast.md) | Report unsupported COPY FROM targets before reading input | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 8 | 2026-11-12 |
| [6932](cf/6932/roast.md) | Improve row estimation with multi-column unique indexes | NEEDS-DESIGN-DISCUSSION | 2 | Paths of one relation get different row counts for the same clauses | 1 | 3 | 2026-11-12 |
| [6943](cf/6943/roast.md) | ON EMPTY clause for aggregate and window functions | NEEDS-DESIGN-DISCUSSION | 4 | No user need shown beyond what COALESCE already gives | 1 | 14 | 2026-11-12 |
| [6954](cf/6954/roast.md) | Proposal: QUALIFY clause | NEEDS-DESIGN-DISCUSSION | 3 | Rewrite design stores views in subquery form, against the reviewer's request | 0 | 40 | 2026-11-12 |
| [6961](cf/6961/roast.md) | REVOKE's CASCADE protection doesn't work with inherited table owners | NEEDS-COMMITTER-EYES | 1 | Back-patched REVOKE behavior change can break existing scripts | 0 | 7 | 2026-11-12 |
| [6968](cf/6968/roast.md) | Tepid: extend the heap-only tuple (HOT) update optimization model such that only those indexes directly impacted by the update need be updated | NEEDS-DESIGN-DISCUSSION | 5 | Collapsed chains can keep pages out of the visibility map indefinitely | 2 | 16 | 2026-11-12 |
| [6972](cf/6972/roast.md) | Re-read subscription state after lock in AlterSubscription | NEEDS-DESIGN-DISCUSSION | 3 | Retry loops in 0003 and 0004 answer a race no user has reported | 2 | 25 | 2026-11-12 |
| [6981](cf/6981/roast.md) | Lazy LEFT NestLoop JOIN: gate rescanning of the inner side by outer-side-only join clauses | NEEDS-DESIGN-DISCUSSION | 4 | Unbounded worst-case cost with no planner choice to avoid it | 1 | 23 | 2026-11-12 |
| [6982](cf/6982/roast.md) | Proposal: INSERT ... BY NAME | NEEDS-COMMITTER-EYES | 1 | Grammar commitments go beyond the unverified draft standard text | 1 | 13 | 2026-11-12 |
| [6984](cf/6984/roast.md) | [PATCH v1 0/7] Wait event timing and tracing instrumentation | NEEDS-DESIGN-DISCUSSION | 3 | Every-transition observation chosen before the observation contract is agreed | 1 | 19 | 2026-11-12 |
| [6991](cf/6991/roast.md) | Add per-backend AIO statistics | NEEDS-DESIGN-DISCUSSION | 5 | v1 grows pgstat_backend.c, a direction the thread already questioned | 2 | 8 | 2026-11-12 |
| [7004](cf/7004/roast.md) | pg_xmin_horizon system view | NEEDS-COMMITTER-EYES | 3 | Second copy of the horizon rules has no test that catches drift | 0 | 30 | 2026-11-12 |
| [7005](cf/7005/roast.md) | Add configure-time choice of either port/atomics.h or <stdatomic.h> for atomic operations | NEEDS-DESIGN-DISCUSSION | 5 | Seq_cst read is a workaround for a bug whose cause is unknown | 0 | 35 | 2026-11-12 |
| [7006](cf/7006/roast.md) | Add bypass IO to pg_stat_io | NEEDS-COMMITTER-EYES | 2 | Counting only in smgr_bulk_flush() leaves other direct smgr writers silently uncounted | 0 | 55 | 2026-11-12 |
| [7008](cf/7008/roast.md) | Separation of OpenSSL and LibreSSL | NEEDS-DESIGN-DISCUSSION | 4 | Two copies of about 1900 shared lines with no check against drift | 1 | 9 | 2026-11-12 |
| [7011](cf/7011/roast.md) | pg_hosts: Add pg_hosts_file_rules() | NO-FUNDAMENTAL-OBJECTION | 0 |  | 2 | 9 | 2026-11-12 |
| [7013](cf/7013/roast.md) | Use Boyer-Moore-Horspool for simple LIKE contains patterns | NEEDS-DESIGN-DISCUSSION | 2 | Low-alphabet data gets slower after upgrade, with no way to opt out | 0 | 14 | 2026-11-12 |
| [7016](cf/7016/roast.md) | Fix LSN of logical decoding message | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 13 | 2026-11-12 |
| [7017](cf/7017/roast.md) | Soft DROP TABLE, recoverable drops for PostgreSQL | NEEDS-DESIGN-DISCUSSION | 1 | Contrib placement adds nothing the external pgtrashcan cannot do | 1 | 11 | 2026-11-12 |
| [7021](cf/7021/roast.md) | Proposal: JSON5 support in the JSON parsers | NEEDS-DESIGN-DISCUSSION | 3 | No user demand shown for a json5 SQL type | 0 | 2 | 2026-11-12 |
| [7023](cf/7023/roast.md) | A new C function `get_partition_root`. | NEEDS-DESIGN-DISCUSSION | 2 | Main demand rests on unmerged EXCEPT patches; two callers today | 0 | 20 | 2026-11-12 |
| [7025](cf/7025/roast.md) | pg_upgrade: add --initdb option to create the new cluster automatically | NEEDS-DESIGN-DISCUSSION | 5 | In-tree versus wrapper question is still open, and the scope has grown | 1 | 28 | 2026-11-12 |
| [7030](cf/7030/roast.md) | Caching a partition index's parent OID in the relcache? | NEEDS-COMMITTER-EYES | 1 | Cache correctness depends on one function remembering to invalidate descendants | 0 | 7 | 2026-11-12 |
| [7043](cf/7043/roast.md) | Make the logical replication conflict messages more like each other | NO-FUNDAMENTAL-OBJECTION | 1 | New reason lines can contradict the detail lines they introduce | 0 | 4 | 2026-11-12 |
| [7055](cf/7055/roast.md) | Collect ALTER PUBLICATION commands for event triggers | NEEDS-COMMITTER-EYES | 3 | Row is reported only when the catalog changed, unlike other ALTER forms | 0 | 11 | 2026-11-12 |
| [7062](cf/7062/roast.md) | pg_upgrade fails when FK constraint with same name exists on partitioned table and its partition | NEEDS-DESIGN-DISCUSSION | 4 | Broad check makes --check fail for clusters that upgrade fine | 1 | 5 | 2026-11-12 |
| [7068](cf/7068/roast.md) | Lost KEY SHARE lock information during toasting allows foreign-key violation | NEEDS-COMMITTER-EYES | 2 | v1 persists wrong xmax and multixact state on disk and on standbys | 0 | 4 | 2026-11-12 |
| [7071](cf/7071/roast.md) | Bug: Whole-row var in indexes corrupts indexes after DDL | NEEDS-COMMITTER-EYES | 3 | New refobjsubid value breaks the assumption that a nonzero subid is a column | 0 | 9 | 2026-11-12 |
| [7074](cf/7074/roast.md) | Introducing find_all_inheritors_ordered() | NEEDS-DESIGN-DISCUSSION | 2 | One caller does not yet justify a shared traversal refactor | 0 | 10 | 2026-11-12 |
| [7079](cf/7079/roast.md) | New data type binary format documentation | NEEDS-COMMITTER-EYES | 2 | Removing testlibpq3 drops the only built example of binary parameters | 0 | 25 | 2026-11-12 |
| [7081](cf/7081/roast.md) | Add PQTRACE env to enable protocol tracing in libpq | NEEDS-COMMITTER-EYES | 4 | trace_file lets dblink and postgres_fdw users write server files | 1 | 2 | 2026-11-12 |
| [7083](cf/7083/roast.md) | Fix write skew under SERIALIZABLE for Tid Scan & Tid Range Scan | NEEDS-DESIGN-DISCUSSION | 2 | TID scan relation lock reverses cb5b28613d5 without addressing its reason | 0 | 14 | 2026-11-12 |
| [7088](cf/7088/roast.md) | Batch fsyncs when recycling WAL segments | NEEDS-DESIGN-DISCUSSION | 2 | Gain is shown only as background checkpoint cleanup time | 1 | 4 | 2026-11-12 |
| [7091](cf/7091/roast.md) | Redesign per-backend statistics | NEEDS-COMMITTER-EYES | 2 | Global view query cost scales with MaxBackends, unmeasured for the array design | 1 | 15 | 2026-11-12 |
| [7097](cf/7097/roast.md) | Release replication slot on error in SQL-callable slot functions | NEEDS-COMMITTER-EYES | 1 | Release on subtransaction abort changes extension-visible slot state in minor releases | 0 | 90 | 2026-11-12 |
| [7102](cf/7102/roast.md) | Parallel heap vacuum | NEEDS-DESIGN-DISCUSSION | 5 | Existing VACUUM commands start heap scan workers without asking | 4 | 137 | 2026-11-12 |
| [7103](cf/7103/roast.md) | Introduce XID age based replication slot invalidation | NEEDS-DESIGN-DISCUSSION | 5 | Primary invalidates a healthy standby's physical slot through forwarded catalog_xmin | 0 | 65 | 2026-11-12 |
| [7105](cf/7105/roast.md) | Add tests for concurrent DML retry paths in logical replication apply | NEEDS-DESIGN-DISCUSSION | 2 | Unmeasured slot cleanup bundled with tests in rarely run apply code | 0 | 5 | 2026-11-12 |
| [7106](cf/7106/roast.md) | Avoid building stderr message if possible | NEEDS-DESIGN-DISCUSSION | 3 | Deferred build at four sites turns a structural invariant into a convention | 1 | 3 | 2026-11-12 |
| [7107](cf/7107/roast.md) | BUG #19603: NaN coordinates in cube type break comparison and GiST scans | NEEDS-COMMITTER-EYES | 2 | Back-patching changes cube btree and GiST ordering in minor releases | 1 | 9 | 2026-11-12 |
| [7117](cf/7117/roast.md) | ALTER COLUMN SET EXPRESSION on partitions not work in case of dependencies | NEEDS-COMMITTER-EYES | 3 | Back-branch fix makes working ALTER TABLE ONLY ... SET EXPRESSION fail | 1 | 6 | 2026-11-12 |
| [7119](cf/7119/roast.md) | Support specialized B-tree page searches | NEEDS-DESIGN-DISCUSSION | 5 | Search control flow copied into per-opclass callbacks in nbtcompare.c | 2 | 3 | 2026-11-12 |
| [7120](cf/7120/roast.md) | Small fixes needed by high-availability tools | NEEDS-DESIGN-DISCUSSION | 5 | Session termination still exposes unreplicated commits without any restart | 3 | 25 | 2026-11-12 |
| [7121](cf/7121/roast.md) | Archive-fed logical decoding: pausing recovery on slot conflict | NEEDS-DESIGN-DISCUSSION | 5 | Some pauses cannot resolve because of WAL ordering, and nothing bounds them | 1 | 3 | 2026-11-12 |
| [7122](cf/7122/roast.md) | FROM clause before SELECT | NEEDS-DESIGN-DISCUSSION | 3 | A psql completion gap is being fixed with server grammar | 1 | 4 | 2026-11-12 |
| [7126](cf/7126/roast.md) | Per-table resync for logical replication subscriptions | NEEDS-COMMITTER-EYES | 3 | Refresh on a forwarding node can stop a downstream subscriber's apply worker | 1 | 36 | 2026-11-12 |
| [7128](cf/7128/roast.md) | DDL deparse | NEEDS-DESIGN-DISCUSSION | 5 | Hand-written deparser keeps the maintenance cost that stopped the earlier patch | 1 | 14 | 2026-11-12 |
| [7131](cf/7131/roast.md) | Fix races in Windows pthread emulation | NEEDS-COMMITTER-EYES | 1 | Ad hoc compiler-specific atomic loads added to two duplicated copies | 0 | 10 | 2026-11-12 |
| [7132](cf/7132/roast.md) | meson: Support ECPG tests in runnable cross builds | NEEDS-COMMITTER-EYES | 2 | Wrapper covers the runner but not the programs it starts | 0 | 3 | 2026-11-12 |
| [7133](cf/7133/roast.md) | Fix pg_basebackup long-path test on Windows | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 4 | 2026-11-12 |
| [7142](cf/7142/roast.md) | enhancing pg_basebackup speeds up to ~23Gbps (small fixes + io_uring/Direct I/O) | NEEDS-DESIGN-DISCUSSION | 5 | New public libpq symbol for a single in-tree consumer | 1 | 15 | 2026-11-12 |
| [7151](cf/7151/roast.md) | Let OpenSSL auto-select DH parameters by default | NEEDS-COMMITTER-EYES | 1 | Silent drop below 2048-bit DH for servers with small keys | 0 | 1 | 2026-11-12 |
| [7161](cf/7161/roast.md) | Apply extended statistics to join clause during parameterized path costing | NEEDS-DESIGN-DISCUSSION | 2 | Repeats the per-relation clausesel.c approach that an earlier attempt was told to restructure | 3 | 3 | 2026-11-12 |
| [7162](cf/7162/roast.md) | Add pg_stat_log_messages: cumulative statistics about server log messages (was: Add contrib module pg_stat_log: cumulative statistics about server log messages) | NEEDS-DESIGN-DISCUSSION | 5 | Unprivileged users can fill the shared table and hide later corruption errors | 2 | 6 | 2026-11-12 |
| [7167](cf/7167/roast.md) | Unlogged materialized views | NEEDS-DESIGN-DISCUSSION | 4 | pg_upgrade may mark a crash-emptied unlogged matview as populated | 1 | 12 | 2026-11-12 |
| [7169](cf/7169/roast.md) | Dropping a composite attribute causes data integrity violations | NEEDS-DESIGN-DISCUSSION | 4 | Accepted DDL becomes an error with no documented migration path | 1 | 3 | 2026-11-12 |
| [7171](cf/7171/roast.md) | Write Combining | NEEDS-COMMITTER-EYES | 5 | Batch checksum on shared lock can write pages with stale checksums | 1 | 43 | 2026-11-12 |
| [7177](cf/7177/roast.md) | Make ginInsertCleanup follow vacuum's BufferAccessStrategy | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 3 | 2026-11-12 |
| [7181](cf/7181/roast.md) | SQL/JSON boolean DEFAULT expression silently replaced with 'false' | NO-FUNDAMENTAL-OBJECTION | 1 | Stored views keep the wrong 'false'::jsonb after the fix, with no detection path | 0 | 5 | 2026-11-12 |
| [7183](cf/7183/roast.md) | Allow pg_read_all_stats to read replication origin status | NEEDS-COMMITTER-EYES | 1 | Same origin progress stays superuser-only through pg_replication_origin_progress() | 0 | 8 | 2026-11-12 |
| [7188](cf/7188/roast.md) | Logical replication row filter loses unchanged toasted columns | NEEDS-COMMITTER-EYES | 5 | New path extends the XLogRecordMaxSize PANIC to tables without REPLICA IDENTITY FULL | 0 | 41 | 2026-11-12 |
| [7190](cf/7190/roast.md) | aio: Async fsyncs for crash recovery and checkpointer | NEEDS-DESIGN-DISCUSSION | 4 | Checkpoint gain is measured with checkpoint_flush_after disabled | 0 | 9 | 2026-11-12 |
| [7192](cf/7192/roast.md) | GiST multirange consistent fix | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 8 | 2026-11-12 |
| [7193](cf/7193/roast.md) | Windows locale and test portability | NEEDS-DESIGN-DISCUSSION | 3 | Test API may become dead weight if frontend locale handling changes | 1 | 5 | 2026-11-12 |
| [7196](cf/7196/roast.md) | Improve cube GiST page splits | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 3 | 2026-11-12 |
| [7197](cf/7197/roast.md) | Duplicate primary keys under SSI | NEEDS-COMMITTER-EYES | 5 | Conflict check loses the deleter once it is summarized | 0 | 7 | 2026-11-12 |
| [7198](cf/7198/roast.md) | Honor fillfactor in sorted GiST builds | NEEDS-DESIGN-DISCUSSION | 2 | Default fillfactor would change every sorted-built GiST index | 0 | 17 | 2026-11-12 |
| [7199](cf/7199/roast.md) | Delete GIN posting-tree pages without excessive locking | NEEDS-DESIGN-DISCUSSION | 1 | No current user is shown to suffer from the root cleanup lock | 0 | 1 | 2026-11-12 |
| [7200](cf/7200/roast.md) | GiST intrapage indexing | NEEDS-DESIGN-DISCUSSION | 5 | The motivating workloads are not measured | 0 | 2 | 2026-11-12 |
| [7203](cf/7203/roast.md) | Persist slot invalidations before publishing them | NEEDS-COMMITTER-EYES | 1 | Backpatch of a new slot locking protocol to 14 without branch versions | 2 | 42 | 2026-11-12 |
| [7206](cf/7206/roast.md) | Fix race with LLVM and bison. | NEEDS-COMMITTER-EYES | 1 | Stated scope covers MODULES only, not MODULE_big or PROGRAM objects | 0 | 5 | 2026-11-12 |
| [7207](cf/7207/roast.md) | Fix timeline history after recovery stops on an ancestor | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 6 | 2026-11-12 |
| [7214](cf/7214/roast.md) | Preserve statistics targets with ALTER TABLE ALTER COLUMN TYPE | NEEDS-COMMITTER-EYES | 1 | Bug fix tag implies a backpatch, but parse node structs change | 0 | 5 | 2026-11-12 |
| [7217](cf/7217/roast.md) | Tighten SCRAM iteration parsing and bound libpq PBKDF2 work | NEEDS-DESIGN-DISCUSSION | 4 | Default scram_max_iterations of 100000 breaks existing high-iteration servers | 1 | 3 | 2026-11-12 |
| [7218](cf/7218/roast.md) | Assorted Win32 error handling fixes (CreateThread, pgwin32_select, pg_usleep) | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 3 | 2026-11-12 |
| [7219](cf/7219/roast.md) | Add PAM Tests and Option For Custom PAM Config Location | NEEDS-DESIGN-DISCUSSION | 2 | pam_wrapper already allows rootless PAM tests without a new pg_hba option | 0 | 3 | 2026-11-12 |
| [7220](cf/7220/roast.md) | pg_rewind: Remove recovery at the start of rewind | NEEDS-DESIGN-DISCUSSION | 5 | Target timeline and control-file fields are read without recovery updating them | 1 | 1 | 2026-11-12 |
| [7221](cf/7221/roast.md) | pg_rewind: file sync bypass and findLastCheckpoint boundary crash | NEEDS-DESIGN-DISCUSSION | 2 | Clean switchover already works by starting the old primary as standby | 1 | 2 | 2026-11-12 |
| [7229](cf/7229/roast.md) | pageinspect: validate line pointers before using them | NEEDS-COMMITTER-EYES | 3 | Whole-page header check takes item listings away from forensic users | 0 | 6 | 2026-11-12 |
| [7232](cf/7232/roast.md) | Protocol compression: a fourth design | NEEDS-DESIGN-DISCUSSION | 5 | Replication streams share one compression history across all users' WAL | 4 | 3 | 2026-11-12 |
| [7237](cf/7237/roast.md) | Reducing relcache memory usage: deduping index shapes | NEEDS-COMMITTER-EYES | 2 | Shared rd_supportinfo changes what support functions may cache in fn_extra | 1 | 31 | 2026-11-12 |
| [7250](cf/7250/roast.md) | glist: _Generic wrapper for selective dlist/dclist usage | NEEDS-DESIGN-DISCUSSION | 3 | New header API with no caller cannot be checked or maintained | 0 | 8 | 2026-11-12 |
| [7253](cf/7253/roast.md) | [PATCH v1 0/1] Avoid carrying self lock-only xmax to updated tuple | NEEDS-COMMITTER-EYES | 1 | Updated rows from ON CONFLICT DO UPDATE now show xmax = 0 | 0 | 6 | 2026-11-12 |
| [7254](cf/7254/roast.md) | Allow bare library names for non-superuser LOAD | NEEDS-DESIGN-DISCUSSION | 2 | Same LOAD string resolves to different files depending on role | 0 | 8 | 2026-11-12 |
| [7255](cf/7255/roast.md) | Fix races in Windows pthread emulation | NEEDS-COMMITTER-EYES | 1 | Compiler-specific atomic loads added to two duplicated mutex copies | 1 | 10 | 2026-11-12 |
| [7257](cf/7257/roast.md) | Fix use-after-free after failed pg_checksum_init | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 6 | 2026-11-12 |
| [7258](cf/7258/roast.md) | Native planner and executor for SQL/PGQ | NEEDS-DESIGN-DISCUSSION | 5 | Native path on by default without row level security enforcement | 0 | 12 | 2026-11-12 |
| [7264](cf/7264/roast.md) | add missed tests for gist index | NEEDS-DESIGN-DISCUSSION | 3 | Recovery TAP test may duplicate replay that 027_stream_regress.pl already does | 3 | 5 | 2026-11-12 |
| [7267](cf/7267/roast.md) | Direct TOAST v2, faster, smaller and no migration needed | NEEDS-DESIGN-DISCUSSION | 5 | Stale direct pointers can read another value's chunk without error | 1 | 31 | 2026-11-12 |
| [7271](cf/7271/roast.md) | Check output_plugin_libraries in pg_createsubscriber | NEEDS-COMMITTER-EYES | 1 | Client-side list parsing can reject settings the server accepts | 0 | 46 | 2026-11-12 |
| [7272](cf/7272/roast.md) | Add recovery boundary WAL record for database and tablespace commands | NEEDS-DESIGN-DISCUSSION | 3 | Boundary record leaves a window where PITR still breaks the catalog | 1 | 4 | 2026-11-12 |
| [7273](cf/7273/roast.md) | Add a permission check to pg_stat_get_backend_subxact() | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 27 | 2026-11-12 |
| [7274](cf/7274/roast.md) | Report changes discarded for relations not in the subscription | NEEDS-COMMITTER-EYES | 1 | Ad hoc LOG line where conflict logging and subscription stats exist | 1 | 4 | 2026-11-12 |
| [7276](cf/7276/roast.md) | Prevent object capture in CREATE/ALTER EXTENSION scripts | NEEDS-COMMITTER-EYES | 4 | Membership in a required extension can be granted by an ordinary role | 1 | 7 | 2026-11-12 |
| [7277](cf/7277/roast.md) | Report userid to PgBackendStatus for standalone backends | NEEDS-DESIGN-DISCUSSION | 1 | No user is shown to need the user name in single-user mode | 0 | 3 | 2026-11-12 |
| [7280](cf/7280/roast.md) | Use maintenance_io_concurrency for parallel index builds | NEEDS-DESIGN-DISCUSSION | 2 | Backpatching changes prefetch depth for existing installations on a minor upgrade | 0 | 5 | 2026-11-12 |
| [7281](cf/7281/roast.md) | Improve Hash/Merge Join estimate accuracy when all predicates are Hash/Merge clauses | NEEDS-DESIGN-DISCUSSION | 3 | No real query shown where the cost change alters the chosen plan | 0 | 7 | 2026-11-12 |
| [7283](cf/7283/roast.md) | Allow subquery pull-up past inlineable CTEs | NEEDS-DESIGN-DISCUSSION | 1 | No real-world query shown that needs a WITH inside a pulled-up subquery | 0 | 6 | 2026-11-12 |
| [7284](cf/7284/roast.md) | {heap,minimal}_expand_tuple() are unused | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 6 | 2026-11-12 |
| [7289](cf/7289/roast.md) | JIT inlining SIGSEGV in MemoryBuffer::getFile with LLVM 17 / GCC 13 | NEEDS-DESIGN-DISCUSSION | 1 | Root cause unconfirmed, so the change may hide a packaging fault | 0 | 1 | 2026-11-12 |
| [7295](cf/7295/roast.md) | [BUG] Fix EXCLUDED virtual generated columns using tableoid | NEEDS-COMMITTER-EYES | 2 | Back-patching a new error breaks statements that run today on v18 | 1 | 11 | 2026-11-12 |
| [7299](cf/7299/roast.md) | Avoid emitting core dumps when not needed | NEEDS-COMMITTER-EYES | 2 | Per-call-site errno marking instead of one rule keyed on the SQLSTATE | 0 | 13 | 2026-11-12 |
| [7302](cf/7302/roast.md) | Optimize MCV stats for sortable types and utilize sorted-order properties | NEEDS-DESIGN-DISCUSSION | 3 | Permanent catalog change and duplicated MCV data for a gain not shown at defaults | 1 | 30 | 2026-11-12 |
| [7304](cf/7304/roast.md) | Discard aborted updaters when expanding a multixact | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 9 | 2026-11-12 |
| [7305](cf/7305/roast.md) | Add get_actual_variable_range index page read limit (complements existing VISITED_PAGES_LIMIT) | NEEDS-COMMITTER-EYES | 1 | Revised limit still gives up early, and no failure-rate data answers message #2 | 0 | 12 | 2026-11-12 |
| [7307](cf/7307/roast.md) | ANSI SQL proposal: SELECT DISTINCT ON (... ORDER BY ...) and UNION DISTINCT ON (... ORDER BY ...) | NEEDS-DESIGN-DISCUSSION | 2 | New DISTINCT ON spellings ship before the standard committee has reviewed them | 1 | 10 | 2026-11-12 |
| [7308](cf/7308/roast.md) | Optimize truncation logic to reduce AccessExclusive lock impact | NEEDS-DESIGN-DISCUSSION | 3 | Wrong relation size from lseek would leave buffers behind | 1 | 5 | 2026-11-12 |
| [7309](cf/7309/roast.md) | Doc: clarify pg_locks descriptions of classid/objid/objsubid | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 2 | 2026-11-12 |
| [7310](cf/7310/roast.md) | Improve hash aggregate spilling by writing only the needed columns | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 5 | 2026-11-12 |
| [7312](cf/7312/roast.md) | BUG #19686: Rolling back SET TABLESPACE | NEEDS-COMMITTER-EYES | 2 | Back-branch ABI change lets an uninitialized field skip the copy | 0 | 33 | 2026-11-12 |
| [7313](cf/7313/roast.md) | Row contention in regress/parallel_schedule, adding multi-line test groups | NEEDS-COMMITTER-EYES | 3 | Indentation-based grammar fixed now, while a known redesign is pending | 0 | 6 | 2026-11-12 |
| [7318](cf/7318/roast.md) | psql single-step mode woes | NEEDS-DESIGN-DISCUSSION | 2 | Backpatching the Ctrl-D fix changes stable-branch behaviour at EOF | 0 | 3 | 2026-11-12 |
| [7324](cf/7324/roast.md) | [Patch] Fix pg_get_multixact_stats() over-reporting members on a hot standby | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 8 | 2026-11-12 |
| [7326](cf/7326/roast.md) | Restructured Shared Buffer Hash Table | NEEDS-DESIGN-DISCUSSION | 4 | Slot-reuse invariant has no agreed safe design, and a miss means duplicate buffers | 1 | 30 | 2026-11-12 |
| [7331](cf/7331/roast.md) | Progress reporting: a debug trace and a test framework | NEEDS-COMMITTER-EYES | 3 | Most checks run only on builds that define PROGRESS_DEBUG | 0 | 2 | 2026-11-12 |
| [7337](cf/7337/roast.md) | pg_walinspect: fix LSN validation messages and empty range handling | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 16 | 2026-11-12 |
| [7338](cf/7338/roast.md) | Use C11 alignas instead of palloc/malloc for alignment | NEEDS-DESIGN-DISCUSSION | 3 | Renaming PGAlignedXLogBlock breaks known extensions for naming consistency only | 0 | 11 | 2026-11-12 |
| [7339](cf/7339/roast.md) | Parameterized append subpaths | NEEDS-COMMITTER-EYES | 2 | Keeping all parameterized paths in add_path() costs every query | 1 | 3 | 2026-11-12 |
| [7340](cf/7340/roast.md) | Speed up lpad() and rpad() | NEEDS-DESIGN-DISCUSSION | 3 | Short-pad callers pay so that large-pad callers gain | 1 | 12 | 2026-11-12 |
| [7344](cf/7344/roast.md) | Fix SSI conflicts between index-only scans and heap writes | NEEDS-COMMITTER-EYES | 1 | Correctness rests on a writer-side ordering rule that nothing enforces | 0 | 3 | 2026-11-12 |
| [7347](cf/7347/roast.md) | Temporary slot leak when creation fails in a subtransaction | NEEDS-COMMITTER-EYES | 1 | Second "creation in progress" mechanism next to the existing ephemeral state | 0 | 7 | 2026-11-12 |
| [7348](cf/7348/roast.md) | Table sync race with REFRESH PUBLICATION | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 6 | 2026-11-12 |
| [7350](cf/7350/roast.md) | pg_dump: Fetch sequence data only for sequences being dumped | NEEDS-COMMITTER-EYES | 1 | Data-OID filter may miss sequences dumped through binary upgrade or extensions | 1 | 2 | 2026-11-12 |
| [7351](cf/7351/roast.md) | Crashes on a partition whose concurrent detach never finished | NEEDS-COMMITTER-EYES | 3 | Back-patching 0004 changes which rows replicate in a minor release | 0 | 37 | 2026-11-12 |
| [7352](cf/7352/roast.md) | Validate GIN posting lists before decoding them | NEEDS-COMMITTER-EYES | 2 | Backpatched replay error can stop standbys and crash recovery after a minor update | 1 | 1 | 2026-11-12 |
| [7353](cf/7353/roast.md) | BUG #19723: CREATE INDEX racing with ALTER INDEX ATTACH PARTITION triggers unexpected internal error | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 4 | 2026-11-12 |
| [7354](cf/7354/roast.md) | Use instr_time for pg_stat_database block read/write time counters | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 13 | 2026-11-12 |
| [7356](cf/7356/roast.md) | [PATCH v1] Fix for Bug#19724 - ALTER TYPE ... ALTER ATTRIBUTE triggers internal error for base type of domain with check | NEEDS-COMMITTER-EYES | 2 | Name-based re-add can attach a constraint to the wrong domain | 0 | 11 | 2026-11-12 |
| [7357](cf/7357/roast.md) | postgres_fdw: Fix costing of remote sorts without remote estimates | NEEDS-COMMITTER-EYES | 1 | Plan changes for existing postgres_fdw users, with no runtime evidence | 0 | 8 | 2026-11-12 |
| [7360](cf/7360/roast.md) | Skip LEFT/ANTI joins to a provably empty inner rel | NEEDS-DESIGN-DISCUSSION | 3 | No real-world query shown that hits an empty inner rel | 0 | 1 | 2026-11-12 |
| [7361](cf/7361/roast.md) | Protocol Compression (fourth attempt) | NEEDS-DESIGN-DISCUSSION | 5 | v2 does not define how users or servers turn compression on or off | 4 | 7 | 2026-11-12 |
| [7362](cf/7362/roast.md) | Halve memory allocation for toast_flatten_tuple | NEEDS-DESIGN-DISCUSSION | 3 | No field report shows this memory peak hurts anyone | 0 | 3 | 2026-11-12 |
| [7364](cf/7364/roast.md) | Costing for parallel scans with few/single row produced in the outer side | NEEDS-DESIGN-DISCUSSION | 5 | Every path constructor must now set effective_workers, and nothing checks it | 0 | 6 | 2026-11-12 |
| [7365](cf/7365/roast.md) | ATTACH PARTITION cost grows linearly with pg_constraint size (seqscan in CloneFkReferenced), much worse since not-null constraints are in pg_constraint (PG 18) | NEEDS-COMMITTER-EYES | 1 | Unreconciled cost of the full index on every constraint insert | 0 | 12 | 2026-11-12 |
| [7366](cf/7366/roast.md) | ReplicationSlotRelease() clobbers another backend's statusFlags entry | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7368](cf/7368/roast.md) | Two fixes for parallel query cleanup: DSM detach order and a stray statement timeout | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [7369](cf/7369/roast.md) | REPACK (CONCURRENTLY): do not block the table while waiting for the final lock | NEEDS-DESIGN-DISCUSSION | 4 | Default REPACK may never finish when lock_timeout is unset | 0 | 4 | 2026-11-12 |
| [7370](cf/7370/roast.md) | BUG #19705: One NaN box makes BRIN and GiST box indexes omit unrelated rows | NEEDS-COMMITTER-EYES | 1 | Two different BRIN fixes for master and back branches | 1 | 16 | 2026-11-12 |
| [7372](cf/7372/roast.md) | Deadlock detector fails to activate on a hot standby replica | NEEDS-COMMITTER-EYES | 1 | Backpatches lag the master fix and miss the lock case | 1 | 43 | 2026-11-12 |
| [7373](cf/7373/roast.md) | synchronized_standby_slots behavior inconsistent with quorum-based synchronous replication | NEEDS-DESIGN-DISCUSSION | 4 | Under ANY, which standby the subscriber can safely fail over to changes at runtime | 0 | 99 | 2026-11-12 |
| [7374](cf/7374/roast.md) | psql: avoid CREATE command completion after GRANT/REVOKE CREATE | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 7 | 2026-11-12 |
| [7376](cf/7376/roast.md) | Reduce LWLockWaitListLock() cache-line contention with adaptive spin reads | NEEDS-COMMITTER-EYES | 1 | Fixed spin constants may not carry over across CPU types, as [P2] found | 2 | 11 | 2026-11-12 |
| [7379](cf/7379/roast.md) | CREATE TABLE .. LIKE copies comments to an unrelated table | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 10 | 2026-11-12 |
| [7380](cf/7380/roast.md) | Fall back to serial vacuum when out of DSM segments | NEEDS-DESIGN-DISCUSSION | 1 | No field report of DSM exhaustion breaking parallel vacuum | 0 | 7 | 2026-11-12 |
| [7382](cf/7382/roast.md) | pg_trgm: don't lose rows in index scans with a zero similarity threshold | NEEDS-COMMITTER-EYES | 1 | Patch fixes indexes to >= while pgtrgm.sgml still documents "greater than" | 0 | 6 | 2026-11-12 |
| [7383](cf/7383/roast.md) | Throwing away unnecessary spin-locks | NEEDS-DESIGN-DISCUSSION | 1 | No measured contention or user problem justifies lock-free conversion | 2 | 13 | 2026-11-12 |
| [7386](cf/7386/roast.md) | standby PANIC on restart after VM truncation | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 11 | 2026-11-12 |
| [7387](cf/7387/roast.md) | Refactor pgbench - pure mechanical modularization into 10 module pairs (stats, variable, poller, random, expr, script, commands, init, engine, report) | NEEDS-DESIGN-DISCUSSION | 2 | No concrete feature or user shown to need this module layout | 2 | 14 | 2026-11-12 |
| [7388](cf/7388/roast.md) | Add ASCII fast path to Unicode normalization functions | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 16 | 2026-11-12 |
| [7390](cf/7390/roast.md) | Enlarge the syslogger pipe on Linux (F_SETPIPE_SZ) | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [7392](cf/7392/roast.md) | Let an ordering index scan hand its ORDER BY value to the target list | NEEDS-DESIGN-DISCUSSION | 5 | Join-level value carrying adds a new path property with no planner owner | 3 | 22 | 2026-11-12 |
| [7394](cf/7394/roast.md) | pg_resetwal: add --cluster-state option | NEEDS-DESIGN-DISCUSSION | 2 | Existing idioms already reset unlogged tables after pg_resetwal | 1 | 5 | 2026-11-12 |
| [7401](cf/7401/roast.md) | gist_trgm_ops '=' operator: planner picks it over btree, ~300x slower | NEEDS-DESIGN-DISCUSSION | 3 | Opclass twin works around a generic GiST costing gap | 0 | 10 | 2026-11-12 |
| [7402](cf/7402/roast.md) | Reduce SyncRepLock contention on the commit path | NEEDS-DESIGN-DISCUSSION | 3 | Deferred release in 0003 may be lost on FATAL or proc_exit paths | 0 | 7 | 2026-11-12 |
| [7403](cf/7403/roast.md) | RLS bypass: ON CONFLICT DO UPDATE/SELECT evaluates WHERE before RLS check | NEEDS-COMMITTER-EYES | 1 | Backpatch turns silent no-op upserts into RLS errors in minor releases | 0 | 1 | 2026-11-12 |
| [7405](cf/7405/roast.md) | Expand OR clauses in joins to UNION ALL paths | NEEDS-DESIGN-DISCUSSION | 5 | No written correctness proof, the gap that stopped the 2017 patch | 2 | 14 | 2026-11-12 |
| [7406](cf/7406/roast.md) | Reset unlogged relations before syncing the data directory? | NEEDS-COMMITTER-EYES | 1 | InitWalRecovery() now runs before the data directory is synced | 1 | 4 | 2026-11-12 |
| [7408](cf/7408/roast.md) | Fix PGTYPESdate_fmt_asc overflow when a year does not fit "yyyy" | NEEDS-COMMITTER-EYES | 1 | Back-branch change to the PGTYPESdate_fmt_asc() failure contract is not defined | 0 | 7 | 2026-11-12 |
| [7410](cf/7410/roast.md) | Planner chooses an index-only scan for an index AM without amcanreturn, and execution fails | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 8 | 2026-11-12 |
| [7411](cf/7411/roast.md) | to_date()/to_timestamp() silently truncate 4+-digit day-of-year input to the first three digits | NEEDS-DESIGN-DISCUSSION | 2 | Behavior change for accepted input needs a stated master-only scope | 2 | 3 | 2026-11-12 |
| [7413](cf/7413/roast.md) | Backend crash (signal 11) in pg_trgm makesign() after ALTER TABLE ... SET STORAGE on a column with a gist_trgm_ops index | NEEDS-COMMITTER-EYES | 1 | No defined repair path for indexes already changed by SET STORAGE | 0 | 4 | 2026-11-12 |
| [7415](cf/7415/roast.md) | Extensible ReadyForQuery wire protocol message and C hook, for connection pools and WAIT FOR LSN | NEEDS-DESIGN-DISCUSSION | 4 | ParameterStatus with GUC_REPORT already pushes session state to clients | 2 | 8 | 2026-11-12 |
| [7417](cf/7417/roast.md) | pgcrypto crypt() can return text with invalid encoding | NEEDS-COMMITTER-EYES | 1 | Back-patching turns working password checks into errors after minor upgrade | 1 | 3 | 2026-11-12 |
| [7418](cf/7418/roast.md) | Remove redundant MultiXactIdIsRunning() check in HeapTupleSatisfiesUpdate() | NO-FUNDAMENTAL-OBJECTION | 0 |  | 1 | 4 | 2026-11-12 |
| [7423](cf/7423/roast.md) | Detoast a column once per row instead of once per reference | NEEDS-DESIGN-DISCUSSION | 4 | In-tree slots promise copy resets that extensions refilling slots in place break | 0 | 3 | 2026-11-12 |
| [7424](cf/7424/roast.md) | Fix out-of-bounds access in pg_bsd_indent's parser stack | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 4 | 2026-11-12 |
| [7425](cf/7425/roast.md) | Unlocked path for GetSnapshotDataReuse | NEEDS-COMMITTER-EYES | 3 | Lockless xmin install in 0003 has no stated argument for hot standby | 3 | 7 | 2026-11-12 |
| [7426](cf/7426/roast.md) | [PG19]Three bugs with a CHECK constraint that only the child enforces | NEEDS-DESIGN-DISCUSSION | 2 | Dump and pg_upgrade turn an inherited child CHECK into a local one | 2 | 16 | 2026-11-12 |
| [7427](cf/7427/roast.md) | [RFC PATCH v1] On-demand WAL replay: accept connections before crash recovery has applied the WAL | NEEDS-DESIGN-DISCUSSION | 5 | No checkpoint during the drain lets pg_wal and the redo window grow | 1 | 2 | 2026-11-12 |
| [7429](cf/7429/roast.md) | LWLock granular partition lock memory layout | NEEDS-DESIGN-DISCUSSION | 2 | No user or workload shown to suffer from padded partition locks | 1 | 5 | 2026-11-12 |
| [7430](cf/7430/roast.md) | bufmgr Half state CAS | NEEDS-DESIGN-DISCUSSION | 1 | Targeted contention shown only on one synthetic single-page workload | 2 | 5 | 2026-11-12 |
| [7432](cf/7432/roast.md) | Fix leak when a plpgsql exception block catches an error from CALL | NEEDS-COMMITTER-EYES | 1 | Correctness depends on two copies of the non-atomic rule agreeing | 0 | 1 | 2026-11-12 |
| [7438](cf/7438/roast.md) | Fix grouping-set nulling of variable-free grouping expressions | NEEDS-DESIGN-DISCUSSION | 1 | Planned matching rules redefine the bug the patch reports | 0 | 8 | 2026-11-12 |
| [7439](cf/7439/roast.md) | Use a hash table for the regexp cache | NEEDS-DESIGN-DISCUSSION | 2 | Groundwork commit has little value until the sizing question is settled | 0 | 3 | 2026-11-12 |
| [7443](cf/7443/roast.md) | pg_upgrade: Prevent premature startup of the new cluster | NEEDS-COMMITTER-EYES | 1 | Marker does not cover the documented rsync standby upgrade | 4 | 4 | 2026-11-12 |
| [7454](cf/7454/roast.md) | Avoid O(N^3) planning time in choose_plan_name() | NEEDS-COMMITTER-EYES | 1 | Two lookup paths must agree, and only rare statements use the hash | 0 | 17 | 2026-11-12 |
| [7456](cf/7456/roast.md) | Fix pg_restore gzip stream validation (bug #19754) | NEEDS-COMMITTER-EYES | 2 | Back-patched bugfix also refactors astreamer_gzip, used by other tools | 1 | 5 | 2026-11-12 |
| [7459](cf/7459/roast.md) | plpgsql: detoast a returned record before keeping it in a non-atomic context | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [7460](cf/7460/roast.md) | Fix double-counting of pending relation stats by VACUUM and ANALYZE | NEEDS-DESIGN-DISCUSSION | 2 | Same cause as an earlier case judged not a bug | 0 | 12 | 2026-11-12 |
| [7462](cf/7462/roast.md) | Improve ndistinct estimates for flattened UNION | NEEDS-DESIGN-DISCUSSION | 2 | Benefit is shown only on synthetic disjoint-key scenarios | 0 | 3 | 2026-11-12 |
| [7464](cf/7464/roast.md) | WITHOUT OVERLAPS foreign key allows referencing EXCLUDE constraint | NEEDS-COMMITTER-EYES | 2 | Back-patched error breaks dump and restore within the 18.x series | 1 | 1 | 2026-11-12 |
| [7465](cf/7465/roast.md) | Fix UNIQUE WITHOUT OVERLAPS + NULLS NOT DISTINCT + multiranges | NEEDS-COMMITTER-EYES | 1 | Backpatched fix gives no path for 18 tables already holding duplicates | 0 | 2 | 2026-11-12 |
| [7466](cf/7466/roast.md) | Fix WITHOUT OVERLAPS PKs used for functional grouping | NEEDS-COMMITTER-EYES | 1 | Stored views that use the temporal key break later dump and upgrade | 1 | 6 | 2026-11-12 |
| [7467](cf/7467/roast.md) | Showing "Table Block Fetches" in EXPLAIN ANALYZE for index-only scans | NEEDS-DESIGN-DISCUSSION | 2 | No one in the thread asks to remove the Heap Fetches line | 0 | 3 | 2026-11-12 |
| [7468](cf/7468/roast.md) | Perf: Specialize ExecIndexOnlyScan() like ExecSeqScan() | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 7 | 2026-11-12 |
| [7470](cf/7470/roast.md) | Use a bulk-write buffer access strategy for TOAST inserts | NEEDS-DESIGN-DISCUSSION | 1 | No measurement shows the TOAST eviction problem in practice | 1 | 2 | 2026-11-12 |
| [7471](cf/7471/roast.md) | ZSTD TOAST compression, and an extensible compression method encoding | NEEDS-DESIGN-DISCUSSION | 5 | Dictionary containment depends on a predicate every future datum path must add | 3 | 48 | 2026-11-12 |
| [7474](cf/7474/roast.md) | Small gaps in IoWorkerMain() error handling | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [7476](cf/7476/roast.md) | Fix plpgsql's R/W optimization for simple expressions run via SPI | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [7477](cf/7477/roast.md) | reindexdb: Fix schema name lookup in parallel mode | NEEDS-COMMITTER-EYES | 1 | Backpatching turns silent successes into failures in minor releases | 0 | 6 | 2026-11-12 |
| [7479](cf/7479/roast.md) | HashAggregate loses worker spill stats on rescan | NEEDS-COMMITTER-EYES | 1 | Rescan semantics fixed for HashAggregate only, other worker-instrumented nodes left unchanged | 1 | 3 | 2026-11-12 |
| [7480](cf/7480/roast.md) | walreceiver: stale hot_standby_feedback xmin on primary when wal_receiver_status_interval = 0 (PG16 and later) | NEEDS-DESIGN-DISCUSSION | 3 | Back-patch makes a minor release invalidate logical slots on standbys | 1 | 4 | 2026-11-12 |
| [7483](cf/7483/roast.md) | Reducing relcache memory usage 2: shrink sizeof(RelationData) | NEEDS-COMMITTER-EYES | 4 | Union reads of the wrong member fail silently in future patches | 2 | 18 | 2026-11-12 |
| [7487](cf/7487/roast.md) | Fix recovery connection detail in EXEC_BACKEND builds | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 4 | 2026-11-12 |
| [7489](cf/7489/roast.md) | GIN jsonb_ops misses root strings in strict jsonpath queries | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 3 | 2026-11-12 |
| [7494](cf/7494/roast.md) | Wrong result for whole-row IS NOT NULL with dropped columns | NEEDS-COMMITTER-EYES | 1 | Back-patched fix leaves expression and partial indexes built under old folding | 0 | 5 | 2026-11-12 |
| [7496](cf/7496/roast.md) | [PATCH 0/3] Improve UNION de-duplication | NEEDS-COMMITTER-EYES | 3 | Implicit LIMIT 1 silently weakens row locking in existing UNION queries | 2 | 1 | 2026-11-12 |
| [7497](cf/7497/roast.md) | pg_stat_statements: fix SQL/JSON behavior keywords and JSON_TABLE query jumbling | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 9 | 2026-11-12 |
| [7498](cf/7498/roast.md) | Use-after-free of a shared Path in add_path()/add_partial_path() | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7499](cf/7499/roast.md) | Optimize mergejoinscansel() by avoiding redundant computations | NEEDS-DESIGN-DISCUSSION | 1 | No evidence that same-column merge self-joins cost real users planning time | 0 | 5 | 2026-11-12 |
| [7501](cf/7501/roast.md) | BUG #19772: ERROR: corrupt MVNDistinct entry for GROUP BY c0, ROLLUP (abs(c0)) with ndistinct statistics | NEEDS-COMMITTER-EYES | 1 | Back-patch changes estimates for queries that did not error | 0 | 2 | 2026-11-12 |
| [7502](cf/7502/roast.md) | Fix rescan costing of single-batch hash joins | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7503](cf/7503/roast.md) | Reject incompatible options specified with VACUUM ONLY_DATABASE_STATS | NEEDS-DESIGN-DISCUSSION | 2 | Rejecting explicit PROCESS_TOAST/PROCESS_MAIN reverses a deliberate tolerance | 1 | 4 | 2026-11-12 |
| [7505](cf/7505/roast.md) | Fix the language for new CREATE SUBSCRIPTION features | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 10 | 2026-11-12 |
| [7507](cf/7507/roast.md) | pgcrypto: Fix sha-crypt rounds= values that do not fit in int (BUG #19743) | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 13 | 2026-11-12 |
| [7511](cf/7511/roast.md) | A bug in recovery prefetcher, triggering asserts / FATAL: checkpoint request failed | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 4 | 2026-11-12 |
| [7512](cf/7512/roast.md) | improving handling of FPIs during recovery prefetching | NEEDS-COMMITTER-EYES | 3 | Lookup cost on every block reference is unmeasured | 0 | 8 | 2026-11-12 |
| [7513](cf/7513/roast.md) | Let's drop support for named POSIX semaphores | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 11 | 2026-11-12 |
| [7519](cf/7519/roast.md) | Fix for bug #19773 : pg_dump differing generation expressions of inherited columns | NEEDS-COMMITTER-EYES | 1 | Patch makes dump/restore produce inconsistent inherited-constraint catalog state | 0 | 8 | 2026-11-12 |
| [7521](cf/7521/roast.md) | Fix recovery conflict canceling the startup progress timeout | NEEDS-COMMITTER-EYES | 1 | Progress timeout stays armed during conflict waits, which wake the startup process | 1 | 3 | 2026-11-12 |
| [7523](cf/7523/roast.md) | nbtree: opt-in leaf key prefix compression | NEEDS-DESIGN-DISCUSSION | 5 | On-disk format would be fixed while its basic layout is still in flux | 2 | 15 | 2026-11-12 |
| [7524](cf/7524/roast.md) | Don't close relation files when the visibility map is extended | NEEDS-DESIGN-DISCUSSION | 3 | Second smgr invalidation kind that callers must choose correctly | 0 | 2 | 2026-11-12 |
| [7525](cf/7525/roast.md) | Support for kqueue in pgbench | NEEDS-DESIGN-DISCUSSION | 4 | Lifting the select() limit may not need a third socket set | 2 | 8 | 2026-11-12 |
| [7526](cf/7526/roast.md) | Print measurements on CTRL-C in pgbench | NEEDS-DESIGN-DISCUSSION | 3 | Existing -T and -P options already cover short manual runs | 0 | 3 | 2026-11-12 |
| [7527](cf/7527/roast.md) | BUG #19737: Empty `JSON_OBJECT` cannot use documented `ON NULL` or unique-key clauses | NEEDS-DESIGN-DISCUSSION | 1 | No use case shown for clauses that cannot change the result | 0 | 10 | 2026-11-12 |
| [7529](cf/7529/roast.md) | Support casts from numeric to oid8 | NEEDS-DESIGN-DISCUSSION | 2 | Quoted literal already covers the full oid8 range | 0 | 1 | 2026-11-12 |
| [7530](cf/7530/roast.md) | Report buffer access strategies in EXPLAIN | NEEDS-COMMITTER-EYES | 2 | Machine-readable EXPLAIN form of the new property is not defined | 1 | 19 | 2026-11-12 |
| [7531](cf/7531/roast.md) | Hash index inserts serialise on the metapage tuple counter | NEEDS-DESIGN-DISCUSSION | 5 | Uncounted tuples persist in hashm_ntuples with no repair on insert-only tables | 0 | 10 | 2026-11-12 |
| [7532](cf/7532/roast.md) | WAL record CRC calculated incorrectly because of underlying buffer modification | NEEDS-COMMITTER-EYES | 2 | No back-branch fix that keeps hash WAL records replayable | 0 | 11 | 2026-11-12 |
| [7533](cf/7533/roast.md) | libpq: Don't read past the end of a message when tracing it | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 3 | 2026-11-12 |
| [7535](cf/7535/roast.md) | BUG #19702: decode() accepts Base64 payload after terminal padding | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7536](cf/7536/roast.md) | JSON_SERIALIZE for JSONB returns parts of the internal JSONB representation | NEEDS-COMMITTER-EYES | 2 | Parser-only fix leaves existing views wrong after a minor update | 0 | 3 | 2026-11-12 |
| [7537](cf/7537/roast.md) | Gather rescan keeps the first scan's tuple bound in workers | NEEDS-COMMITTER-EYES | 1 | Back-branch fix changes the signature of an exported executor function | 0 | 2 | 2026-11-12 |
| [7538](cf/7538/roast.md) | GIN index fast list search may become un-interruptible for long time | NEEDS-COMMITTER-EYES | 2 | Speedup leaves the uncancellable loop in place for other workloads | 0 | 4 | 2026-11-12 |
| [7541](cf/7541/roast.md) | BUG #19703: information_schema.usage_privileges omits a sequence owner's implicit USAGE privilege | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7542](cf/7542/roast.md) | Reject unassigned bytes in single-byte encodings | NEEDS-DESIGN-DISCUSSION | 5 | pg_upgrade carries forward bytes that dump and restore will then reject | 0 | 1 | 2026-11-12 |
| [7544](cf/7544/roast.md) | Add amnotifydelete, an index AM callback for deleted tuples | NEEDS-DESIGN-DISCUSSION | 2 | Row-level triggers already deliver old values within the deleting transaction | 1 | 4 | 2026-11-12 |
| [7545](cf/7545/roast.md) | BUG #19752: GROUP BY on a constant and a cast to a length-limited array type crashes the server while planning | NEEDS-COMMITTER-EYES | 1 | Views already stored on 18.x keep the bad substituted tree after the fix | 0 | 11 | 2026-11-12 |
| [7546](cf/7546/roast.md) | pgstat: Add helper to drop entries from a stats kind's hash | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 18 | 2026-11-12 |
| [7548](cf/7548/roast.md) | Fall back to serializing when parallel apply runs out of DSM segments | NO-FUNDAMENTAL-OBJECTION | 0 |  | 2 | 5 | 2026-11-12 |
| [7550](cf/7550/roast.md) | Check the fd number against FD_SETSIZE in PQsocketPoll | NEEDS-COMMITTER-EYES | 1 | Guard goes into a branch no supported Unix may compile | 0 | 2 | 2026-11-12 |
| [7552](cf/7552/roast.md) | Setting the number of digits after the decimal point with a psql's pset variable | NEEDS-DESIGN-DISCUSSION | 5 | Truncation makes psql output disagree with round() and with stored values | 1 | 7 | 2026-11-12 |
| [7553](cf/7553/roast.md) | Fix cumulative WAL statistics double-counting from parallel workers | NEEDS-COMMITTER-EYES | 3 | Unenforced rule: every pgWalUsage diff for cumulative stats must subtract workers | 2 | 24 | 2026-11-12 |
| [7554](cf/7554/roast.md) | Assert failure in parallel query due to SubPlan double visit for hash join | NEEDS-COMMITTER-EYES | 2 | Proposed walker change cannot reach the back branches that have the bug | 0 | 8 | 2026-11-12 |
| [7556](cf/7556/roast.md) | Uncancellable infinite loop in xpath() with entity references | NEEDS-COMMITTER-EYES | 3 | Bugfix tag conflicts with a fix that changes query results | 2 | 1 | 2026-11-12 |
| [7558](cf/7558/roast.md) | Support "ALTER FOREIGN TABLE ... RENAME CONSTRAINT ..." | NEEDS-DESIGN-DISCUSSION | 1 | Rename constraint on foreign tables already works through alter table | 0 | 1 | 2026-11-12 |
| [7560](cf/7560/roast.md) | Proposal: Include CHECK constraint expression in violation error DETAIL | NEEDS-DESIGN-DISCUSSION | 2 | No demand shown beyond the author's own wish | 1 | 4 | 2026-11-12 |
| [7561](cf/7561/roast.md) | Call CalculateShmemSize() only once | NEEDS-COMMITTER-EYES | 1 | Backpatching to PG 19 changes exported API and startup behaviour in a stable branch | 0 | 7 | 2026-11-12 |
| [7563](cf/7563/roast.md) | Faulty pg_get_shmem_pagesize() on Windows | NEEDS-COMMITTER-EYES | 1 | Recorded page size has no check when an allocation path skips it | 0 | 1 | 2026-11-12 |
| [7565](cf/7565/roast.md) | Quickselect instead of a heap for ORDER BY ... LIMIT N large bounded sorts | NEEDS-COMMITTER-EYES | 3 | Regression on ORDER BY ... DESC LIMIT 1 over append-ordered tables | 0 | 5 | 2026-11-12 |
| [7566](cf/7566/roast.md) | Add an ETW backend for the static probes on Windows | NEEDS-DESIGN-DISCUSSION | 5 | Default-on ETW exposes all sessions' query text to Performance Log Users | 0 | 1 | 2026-11-12 |
| [7568](cf/7568/roast.md) | pg_upgrade doesn't copy ICU rules on template0 | NEEDS-COMMITTER-EYES | 1 | Back-branch fix bundles unrelated pg_upgrade refactoring | 0 | 7 | 2026-11-12 |
| [7569](cf/7569/roast.md) | Fix trigger memory leak | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7570](cf/7570/roast.md) | Reject wholerow references where ColumnRef is forbidden | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 3 | 2026-11-12 |
| [7571](cf/7571/roast.md) | UPDATE/DELETE FOR PORTION OF | NEEDS-COMMITTER-EYES | 3 | PERIOD DDL adds hidden columns with no dump, psql or catalog story | 1 | 68 | 2026-11-12 |
| [7572](cf/7572/roast.md) | Standby stuck after walreceiver restart just before a WAL segment boundary | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7573](cf/7573/roast.md) | Fix pg_combinebackup with truncated intermediate incremental backups | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7574](cf/7574/roast.md) | psql: Don't offer USING INDEX after REPACK (CONCURRENTLY) or (ANALYZE) | NEEDS-COMMITTER-EYES | 1 | Tab completion copies a server-side semantic rule that can drift | 0 | 1 | 2026-11-12 |
| [7575](cf/7575/roast.md) | contrib/fuzzystrmatch: improve test coverage | NO-FUNDAMENTAL-OBJECTION | 0 |  | 6 | 1 | 2026-11-12 |
| [7577](cf/7577/roast.md) | Stale hash entries when reusing an auxiliary resource owner | NEEDS-COMMITTER-EYES | 2 | Stable-branch target, but v4 adds a public API to resowner.h | 0 | 7 | 2026-11-12 |
| [7578](cf/7578/roast.md) | Periodically refresh database frozen IDs during autovacuum | NEEDS-COMMITTER-EYES | 2 | vacuumdb runs with SKIP_DATABASE_STATS and gets no periodic refresh | 0 | 8 | 2026-11-12 |
| [7579](cf/7579/roast.md) | Fix ReadStream handling when heap rescan strategy changes | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 6 | 2026-11-12 |
| [7580](cf/7580/roast.md) | doc: Fix description of relative extension "directory" setting | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 7 | 2026-11-12 |
| [7581](cf/7581/roast.md) | extension_control_path: ignore empty elements and deduplicate directories | NEEDS-DESIGN-DISCUSSION | 2 | Rejecting empty and relative elements breaks settings that work today | 0 | 9 | 2026-11-12 |
| [7587](cf/7587/roast.md) | pg_combinebackup: Don't point only to pg_checksums for mixed checksum states | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 44 | 2026-11-12 |
| [7589](cf/7589/roast.md) | Fix exclude generated columns in COPY TO with RLS | NEEDS-DESIGN-DISCUSSION | 1 | Backpatch would change COPY TO output columns in a minor release | 0 | 3 | 2026-11-12 |
| [7590](cf/7590/roast.md) | BUG #19689: MERGE INSERT accepts a set-returning function during PREPARE but fails at EXECUTE | NEEDS-COMMITTER-EYES | 1 | Ad hoc flag check instead of the expression-kind mechanism for SRF placement | 0 | 2 | 2026-11-12 |
| [7591](cf/7591/roast.md) | BUG? Unpublished generated columns are reported "missing" by the subscription | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 7 | 2026-11-12 |
| [7592](cf/7592/roast.md) | Assertion failure in WAL summarizer after promotion | NEEDS-COMMITTER-EYES | 2 | Backpatch scope for the data-loss fixes 0002 and 0003 is not stated | 0 | 13 | 2026-11-12 |
| [7593](cf/7593/roast.md) | Fix initial sync of zero-column published tables | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 7 | 2026-11-12 |
| [7595](cf/7595/roast.md) | MERGE fails to reject source references in NOT MATCHED BY SOURCE actions | NEEDS-COMMITTER-EYES | 1 | Back-patch policy for newly rejected MERGE queries is not stated | 0 | 1 | 2026-11-12 |
| [7598](cf/7598/roast.md) | Improve row mark handling in self-join elimination | NEEDS-DESIGN-DISCUSSION | 1 | No user is shown writing self-joins with differing lock strengths | 1 | 7 | 2026-11-12 |
| [7599](cf/7599/roast.md) | doc: Mention that * does not include system columns | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 10 | 2026-11-12 |
| [7600](cf/7600/roast.md) | "variable not found in subplan target list", DISTINCT agg over Gather` | NEEDS-COMMITTER-EYES | 2 | Conservative early-sort check may change plans for working queries | 0 | 5 | 2026-11-12 |
| [7601](cf/7601/roast.md) | BUG #19798: "failed to build any 4-way joins" with ROLLUP over a LATERAL subquery that references the nullable s | NEEDS-COMMITTER-EYES | 1 | Local pruning exception duplicates the meaning of has_join_restriction() | 0 | 2 | 2026-11-12 |
| [7602](cf/7602/roast.md) | Propagate LIMIT bound through the outer side of left join | NEEDS-DESIGN-DISCUSSION | 2 | Correctness rests on an unenforced fetch-order property of join nodes | 1 | 2 | 2026-11-12 |
| [7604](cf/7604/roast.md) | Include schema-qualified names in publication error messages (generalised to all of LR). | NEEDS-COMMITTER-EYES | 3 | Two relation-name helpers with opposite quoting rules | 2 | 39 | 2026-11-12 |
| [7605](cf/7605/roast.md) | Validate JSON object keys before omitting null values (BUG #19736) | NEEDS-DESIGN-DISCUSSION | 2 | Queries that return {} today start failing, with no back-patch or release-note plan | 2 | 8 | 2026-11-12 |
| [7606](cf/7606/roast.md) | EXPLAIN ANALYZE overcounts MERGE inserts suppressed by triggers | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [7608](cf/7608/roast.md) | Add pg_stat_progress overview view | NEEDS-DESIGN-DISCUSSION | 1 | No demand shown beyond the author's wish for one overview | 0 | 1 | 2026-11-12 |
| [7609](cf/7609/roast.md) | XMLTable treats a column of a domain over xml as a non-xml column | NEEDS-COMMITTER-EYES | 1 | Back-patching changes results of existing XMLTABLE queries in minor releases | 0 | 1 | 2026-11-12 |
| [7611](cf/7611/roast.md) | psql: Fix REPACK index completion after a column list | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 5 | 2026-11-12 |
| [7612](cf/7612/roast.md) | Clarify wait versus poll behavior in ExecAppendAsyncEventWait | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 6 | 2026-11-12 |
| [7613](cf/7613/roast.md) | Timing issue in pg_basebackup | NEEDS-COMMITTER-EYES | 1 | Client-side reordering leaves the race for other BASE_BACKUP clients | 0 | 6 | 2026-11-12 |
| [7614](cf/7614/roast.md) | XMLConcat() and XMLRoot() silently drop xml values with an invalid declaration | NEEDS-COMMITTER-EYES | 1 | New errors in a stable-branch bugfix need a stated back-patch scope | 0 | 1 | 2026-11-12 |
| [7615](cf/7615/roast.md) | Eliminate redundant outer-join clone clauses after constant propagation | NEEDS-DESIGN-DISCUSSION | 2 | Plan-changing optimization framed around back-branch ABI limits | 1 | 1 | 2026-11-12 |
| [7617](cf/7617/roast.md) | Keep TIDBitmap pagetable memory across bitmap scans | NEEDS-DESIGN-DISCUSSION | 2 | Most of the gain is available today through glibc malloc environment variables | 1 | 3 | 2026-11-12 |
| [7618](cf/7618/roast.md) | Backend crash in expanded_record_info function | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [7619](cf/7619/roast.md) | Avoid redundant BitmapAnd with a clauseless partial index | NEEDS-COMMITTER-EYES | 1 | Extra predicate proofs add planning cost for every partial-index query | 2 | 1 | 2026-11-12 |
| [7620](cf/7620/roast.md) | Fix interval input at minimum signed field values | NEEDS-COMMITTER-EYES | 1 | v2 breaks interval inputs that master accepts when the minimum field comes last | 0 | 2 | 2026-11-12 |
| [7621](cf/7621/roast.md) | Fix jsonpath printing of operations followed by an accessor | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 1 | 2026-11-12 |
| [7622](cf/7622/roast.md) | XMLElement() and XMLForest() embed the XML declaration of xml arguments | NEEDS-COMMITTER-EYES | 1 | Back-patching changes xmlelement() and query_to_xml() output in stable branches | 0 | 1 | 2026-11-12 |
| [7623](cf/7623/roast.md) | Injection point: fix race condition between attach and cleanup | NEEDS-DESIGN-DISCUSSION | 2 | New core API has one consumer while a test-only fix exists | 1 | 3 | 2026-11-12 |
| [7624](cf/7624/roast.md) | Match jsonpath variable names exactly when checking mutability | NEEDS-COMMITTER-EYES | 1 | Back-patch makes existing dumps and pg_upgrade fail without a way to find them | 0 | 1 | 2026-11-12 |
