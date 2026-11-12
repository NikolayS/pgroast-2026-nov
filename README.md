# Design pre-reviews of CommitFest entries

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

Each entry was read from its mailing-list thread and the prose of its latest patch (commit message, file list, documentation changes). No code was read. A report lists objections that a reviewer may raise, with links to the thread. It is one-sided on purpose and it is not a decision about the patch. The verdict line is the least reliable part: read the objections.

Made with [pgroast](https://github.com/NikolayS/pgroast). In each folder: `roast.md` (short report), `roast-full.md` (everything, with the models used), `roast.json`, `roast-summary.txt`.

Entries done: 114.

- NEEDS-DESIGN-DISCUSSION: 53
- NEEDS-COMMITTER-EYES: 44
- NO-FUNDAMENTAL-OBJECTION: 16
- LIKELY-REJECT: 1

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
| [5621](cf/5621/roast.md) | Compress big WAL records | NEEDS-DESIGN-DISCUSSION | 5 | Gain for whole-record compression is shown on one synthetic workload only | 0 | 37 | 2026-11-12 |
| [5631](cf/5631/roast.md) | Allow table AMs to define their own reloptions | NEEDS-COMMITTER-EYES | 5 | Thread does not explain how v14 avoids the 2024 revert reasons | 4 | 39 | 2026-11-12 |
| [5664](cf/5664/roast.md) | Fix slot synchronization with two_phase decoding enabled | NEEDS-DESIGN-DISCUSSION | 1 | Minor release makes dumps from older 17.x releases fail to restore | 1 | 100 | 2026-11-12 |
| [5667](cf/5667/roast.md) | support create index on virtual generated column. | NEEDS-DESIGN-DISCUSSION | 2 | Expression indexes on the generation expression already meet the stated need | 0 | 28 | 2026-11-12 |
| [5671](cf/5671/roast.md) | Produce LLVM bitcode files on meson builds | NEEDS-DESIGN-DISCUSSION | 5 | General extension .pc file has no demonstrated users yet | 0 | 32 | 2026-11-12 |
| [5774](cf/5774/roast.md) | Extend skipping FK checks on replicas to include ADD FK and TRUNCATE | NEEDS-DESIGN-DISCUSSION | 3 | Earlier objection to extending replica-role FK bypass to truncate is unanswered | 1 | 12 | 2026-11-12 |
| [5845](cf/5845/roast.md) | Carefully exposing information without authentication | LIKELY-REJECT | 3 | A small separate service already answers this without core changes | 0 | 18 | 2026-11-12 |
| [5877](cf/5877/roast.md) | Report bytes and transactions actually sent downtream | NEEDS-COMMITTER-EYES | 2 | One output_bytes column counts different framing per consumer path | 0 | 123 | 2026-11-12 |
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
| [6177](cf/6177/roast.md) | Add archive_mode to prevent unarchived WAL on permanent primary failure | NEEDS-DESIGN-DISCUSSION | 5 | Ancestor-timeline segments marked .done without proof they are archived | 1 | 42 | 2026-11-12 |
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
| [6440](cf/6440/roast.md) | tablecmds: fix bug where index rebuild loses replica identity on partitions | NEEDS-DESIGN-DISCUSSION | 4 | Two competing mechanisms for the same rebuild path | 0 | 45 | 2026-11-12 |
| [6763](cf/6763/roast.md) | Improve pg_stat_statements scalability | NEEDS-COMMITTER-EYES | 5 | Removing pg_stat_statements could discard all cumulative statistics at restart | 5 | 29 | 2026-11-12 |
| [6765](cf/6765/roast.md) | teach vacuumlo to handle domains | NEEDS-COMMITTER-EYES | 2 | Domain fix and composite/array expansion are bundled in one patch | 0 | 7 | 2026-11-12 |
| [6781](cf/6781/roast.md) | pgstat: Flush some statistics within running transactions, take 2 | NEEDS-COMMITTER-EYES | 2 | Custom stats callbacks can silently double count after the API change | 1 | 44 | 2026-11-12 |
| [7016](cf/7016/roast.md) | Fix LSN of logical decoding message | NO-FUNDAMENTAL-OBJECTION | 0 |  | 0 | 13 | 2026-11-12 |
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
