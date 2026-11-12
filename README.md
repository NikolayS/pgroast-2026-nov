# Design pre-reviews of CommitFest entries

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

Each entry was read from its mailing-list thread and the prose of its latest patch (commit message, file list, documentation changes). No code was read. A report lists objections that a reviewer may raise, with links to the thread. It is one-sided on purpose and it is not a decision about the patch. The verdict line is the least reliable part: read the objections.

Made with [pgroast](https://github.com/NikolayS/pgroast). In each folder: `roast.md` (short report), `roast-full.md` (everything, with the models used), `roast.json`, `roast-summary.txt`.

Entries done: 54.

- NEEDS-COMMITTER-EYES: 27
- NO-FUNDAMENTAL-OBJECTION: 14
- NEEDS-DESIGN-DISCUSSION: 13

| Entry | Title | Verdict | Objections | First objection | Prior attempts | Messages read | Date |
|---|---|---|---:|---|---:|---:|---|
| [4351](cf/4351/roast.md) | Index Prefetching | NEEDS-COMMITTER-EYES | 4 | GiST batches on a hot standby can return deleted tuples | 0 | 515 | 2026-11-12 |
| [4460](cf/4460/roast.md) | Implement row pattern recognition feature | NEEDS-COMMITTER-EYES | 5 | Reduced-frame design is still open and would change the window API | 0 | 549 | 2026-11-12 |
| [4573](cf/4573/roast.md) | pg_ctl start may return 0 even if the postmaster has been already started on Windows | NEEDS-COMMITTER-EYES | 4 | Dropping cmd.exe brings back the AutoRun change reverted for v17 | 0 | 43 | 2026-11-12 |
| [4681](cf/4681/roast.md) | Make COPY format extendable: Extract COPY TO format implementations | NEEDS-COMMITTER-EYES | 5 | Every new COPY option needs a decision in every custom format | 1 | 341 | 2026-11-12 |
| [4817](cf/4817/roast.md) | COPY (on_error table) saving error info to a table | NEEDS-DESIGN-DISCUSSION | 5 | Fixed pg_catalog row type makes later column changes break upgrades | 2 | 31 | 2026-11-12 |
| [4952](cf/4952/roast.md) | Use WALReadFromBuffers in more places | NEEDS-COMMITTER-EYES | 2 | Buffer-only reads leave seg.ws_tli unset, and the timeline invariant is not stated | 0 | 31 | 2026-11-12 |
| [4971](cf/4971/roast.md) | [CREATE\|RE] INDEX CONCURRENTLY with single heap scan | NEEDS-DESIGN-DISCUSSION | 5 | Validation snapshot refresh needs a stated argument against the PG14 failure | 3 | 126 | 2026-11-12 |
| [5018](cf/5018/roast.md) | Extension security improvement: Add support for extensions with an owned schema | NEEDS-DESIGN-DISCUSSION | 5 | Owned schema shared via CASCADE breaks restore and pg_upgrade drops a dependency | 0 | 43 | 2026-11-12 |
| [5029](cf/5029/roast.md) | Historic snapshot doesn't track txns committed in BUILDING_SNAPSHOT state | NEEDS-COMMITTER-EYES | 1 | Hand-kept list of catalog-marking WAL records in the BUILDING_SNAPSHOT path | 0 | 10 | 2026-11-12 |
| [5129](cf/5129/roast.md) | Asynchronous MergeAppend Execution | NEEDS-COMMITTER-EYES | 2 | Async FDWs now receive MergeAppendState with no opt-in or loud failure | 0 | 64 | 2026-11-12 |
| [5151](cf/5151/roast.md) | DirtySnapshot index scan skips concurrently updated tuples leading to lost updates/deletes in logical replication | NEEDS-COMMITTER-EYES | 2 | MVCC lookup drops the wait for an in-flight local INSERT | 0 | 45 | 2026-11-12 |
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
