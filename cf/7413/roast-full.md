# Design pre-review: Backend crash (signal 11) in pg_trgm makesign() after ALTER TABLE ... SET STORAGE on a column with a gist_trgm_ops index

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7413](https://commitfest.postgresql.org/patch/7413/) (Needs review); authors: Andrey Rachitskiy (a.rachitskiy)
- Thread: [Backend crash (signal 11) in pg_trgm makesign() after ALTER TABLE ... SET STORAGE on a column with a gist_trgm_ops index](https://www.postgresql.org/message-id/flat/CAA8TiqELrCMyAkNMev0NerxACZTaQL5a2Y2htzfDxFirJ%2BQGkw%40mail.gmail.com), started 2026-10-02
- Messages read: 4 of 4
- Patch prose read: none (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated with a reproducer and a backtrace, and demand is shown: a backend crash on 16, 17 and 18. No objection survived at level 1a or 1b. The one surviving objection, standards-compat-1, is a fixable level 2 point: v1 stops new damage but the thread does not settle how users repair indexes that SET STORAGE already changed. A committer who knows index catalog code needs to decide the repair path and the back-branch plan.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: No patch was readable, so the exact type comparison, the SET COMPRESSION behaviour and the tests could not be checked. The REINDEX POC patch and message #4 were also not visible in full.

## The proposal, restated

**Problem.** alter table ... alter column ... set storage EXTENDED or MAIN copies the new attstorage onto index columns whose type is the opclass STORAGE type, such as gtrgm. Later inserts write short varlena headers, and code that reads a 4-byte header crashes the backend. The reporter hit this on 16, 17 and 18, and on managed cloud services.

**Proposed design.** SetIndexStorageProperties() skips the catalog update when the index column type differs from the heap column type, so a btree index on the same column still gets the new setting. SET COMPRESSION uses the same helper and gets the same rule. A second POC patch from another reviewer makes REINDEX reset wrong attstorage values on indexes that were already changed.

**User-visible surface.**
- alter table ... set storage no longer changes pg_attribute.attstorage of GiST, GIN or SP-GiST index columns whose type comes from the opclass STORAGE type.
- alter table ... set compression follows the same rule for those index columns.
- REINDEX resets wrong attstorage values on affected indexes (POC patch only).

**What the author claims.**
- The bug is that the helper copied heap attstorage even after ConstructTupleDescriptor() replaced the index column type with the opclass STORAGE type.
- A btree index on the same column still receives the new setting.
- Kirill Reshke claims btree, hash and BRIN are safe because those AMs reject a STORAGE type that differs from the data type.
- Kirill Reshke shows lost rows in index scans with intarray gist__intbig_ops, and says third-party SP-GiST opclasses can crash the same way.
- Kirill Reshke says REINDEX fails on an affected index in the same way as the insert.

**Thread state.** Andrey Rachitskiy posted a v1 fix the same day, and Kirill Reshke widened the scope to GIN, GiST and SP-GiST and added a REINDEX POC patch. Nobody has reviewed either patch yet, and CFbot reports that the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-02; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No defined repair path for indexes already changed by SET STORAGE

fixable · level 2, rubric item 14 · backpatchability, catalog changes · `standards-compat-1`

A user on 16 ran SET STORAGE EXTENDED last year, and their gist_trgm_ops index column still has attstorage 'x'. After the minor update with v1, the next insert still crashes, or an intarray index scan still loses rows, until someone repairs that index.

The v1 fix in message #2 only stops new damage, and the reporter found the bug on 16, 17, 18 and managed services, so damaged catalogs already exist in the field. Message #1 says REINDEX resets the column to 'p'. Message #4 says REINDEX fails the same way and adds a separate POC patch to reset the catalog values. The thread does not say whether the POC patch ships with v1, or what the release notes tell users to run.

Thread: [#1 (Thiago Bonfante, 2026-10-02)](https://www.postgresql.org/message-id/CAA8TiqELrCMyAkNMev0NerxACZTaQL5a2Y2htzfDxFirJ%2BQGkw%40mail.gmail.com), [#2 (Andrey Rachitskiy, 2026-10-02)](https://www.postgresql.org/message-id/CAB8bMitfqZY32oVmB2ZgSRFA68HwpBYD5sKHtL0uKrwMyHNmMQ%40mail.gmail.com), [#4 (Kirill Reshke, 2026-10-04)](https://www.postgresql.org/message-id/CALdSSPjdKwQAX%2BQJd7%2BrcorAZd1GkqKZuTbjciYj8QpiKjf6%2Bg%40mail.gmail.com)

**Question:** Does REINDEX on a damaged index work on released branches, and if not, does the reset patch go into every back branch together with v1?

Counter-check: held. Message #1 says REINDEX resets the column to 'p', and message #4 says REINDEX fails the same way. The thread does not settle this, and it does not say whether the REINDEX POC patch ships with v1.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Affected indexes already hold keys with short varlena headers, and v1 leaves those index pages as they are. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The fix changes when SetIndexStorageProperties() updates pg_attribute.attstorage for index columns. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Today a table owner can run SET STORAGE and make later inserts crash the backend; the fix removes that path. |
| backpatchability | **yes** | The crash is reported on 16, 17 and 18, and the proposal does not say which branches get the fix. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The REINDEX POC patch adds a catalog reset step in REINDEX that must stay in step with the create-time storage rule. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does REINDEX succeed on an index whose attstorage SET STORAGE already changed, given that message #1 says it does and message #4 says it fails?
2. Do the v1 fix and the REINDEX POC patch go in together, and into which back branches?
3. What exact step should the release notes tell users to run to repair an already affected index?
4. Do pg_upgrade and pg_dump carry the wrong attstorage, or the short-header keys, of an affected index to the new cluster?
5. Is comparing the index column type with the heap column type enough for every GiST, GIN and SP-GiST opclass with a STORAGE type, including third-party ones?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows ALTER TABLE catalog updates in tablecmds.c and how index tuple descriptors take the opclass STORAGE type; no such committer has spoken in the visible thread.
- **Liability:** A back-patched change to which pg_attribute rows SET STORAGE and SET COMPRESSION update, plus the repair advice in the release notes. If the REINDEX POC patch goes in, the owner also signs for a catalog reset inside REINDEX on all supported branches.
- **Author follow-up:** Andrey Rachitskiy posted v1 the same day as the report and created the CommitFest entry. CFbot reports that the patch needs a rebase, and no rebase is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** This is a reproducible backend crash and, per message #4, lost rows in index scans, for users of GiST, GIN or SP-GiST opclasses with a STORAGE type. The fix is one rule in one helper with no new user-visible surface; review cost is medium because no patch was read and the repair path is open.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` No repair path for indexes already holding short-header keys. Refuted (not specific): Duplicate of standards-compat-1. Its pg_upgrade point is plausible from general Postgres knowledge, but the thread does not test it and it asks the author for the same repair step.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] concerns storage settings under table inheritance and pg_dump round-trips, and [P3] concerns changing opclass options without a rebuild; neither recorded reason argues against limiting SET STORAGE on opclass storage-type index columns. The thread points to no earlier discussion of this bug.
- standards-compat: 1 objection(s). The core change restores the documented contract, since index columns of opclass STORAGE types keep their type's storage, and it needs no new syntax, GUC or dump change. pg_dump creates indexes after table storage settings, so dump and pg_upgrade likely rebuild clean values, but I could not check this without the patch.
- replication-ondisk: 1 objection(s). The fix itself adds no on-disk format, WAL record, protocol or catalog column change, so only the existing-damage path is a concern from this angle.
- security: 0 objection(s). The fix narrows exposure. Today any table owner can make later inserts crash the backend, and every session restarts; the patch removes that path and adds no new privilege, role or exposed data. Indexes that are already broken still crash on insert and on REINDEX, but that is a repair and backpatch question for other angles.
- maintenance-fit: 0 objection(s). No maintenance or fit objection applies: this is a crash and wrong-result fix in core ALTER TABLE code, with no new mechanism, GUC, catalog field or hot-path cost. The open points (whether REINDEX heals affected indexes, given that message #1 and message #4 disagree, which branches get the fix, and whether the create-time and alter-time storage rules should share one code path) are review questions for the patch, not reasons to reject it.
- already-exists: 0 objection(s). The proposal fixes a backend crash and wrong index scan results. The only existing alternative is a workaround, DROP INDEX plus CREATE INDEX, and it does not stop SET STORAGE from setting the wrong attstorage again. No extension or existing setting can prevent the catalog update, so from this angle the fix must be in core.
- demand: 0 objection(s). Demand is shown. This is a reproducible backend crash on a stock install, reported against 16, 17 and 18 and first seen on two managed services (message #1). Message #4 adds wrong index scan results with intarray, so every user of these GiST opclasses is exposed, and the fix only changes catalog writes for opclass storage-type index columns.

### Precedents retrieved

- P1: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - used in the review; reason summarised from its thread
- P2: [Vertical Clustered Index (columnar storage extension)](https://commitfest.postgresql.org/patch/945/) (cf, Returned with feedback, 2017-03-06) - judged not relevant; reason summarised from its thread
- P3: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - used in the review; reason summarised from its thread
- P4: [Columnar storage](https://commitfest.postgresql.org/patch/354/) (cf, Returned with feedback, 2016-01-31) - judged not relevant
- P5: [Custom storage managers (SMGR), redux](https://commitfest.postgresql.org/patch/4428/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P6: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant; reason summarised from its thread
- P7: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P8: [Extensible storage manager API - smgr hooks](https://commitfest.postgresql.org/patch/3216/) (cf, Returned with feedback, 2022-03-22) - judged not relevant; reason summarised from its thread
- P9: [Tab completion for ALTER COLUMN SET STATISTICS](https://commitfest.postgresql.org/patch/377/) (cf, Rejected, 2015-12-21) - judged not relevant; reason summarised from its thread
- P10: [Clarify the comments about varlena header encoding](https://commitfest.postgresql.org/patch/3820/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P11: [Parallel CREATE INDEX for GiST indexes](https://commitfest.postgresql.org/patch/5026/) (cf, Returned with feedback, 2025-02-06) - judged not relevant
- P12: [postgres.exe doesn't output crash dump when it crashes before main()](https://commitfest.postgresql.org/patch/1525/) (cf, Rejected, 2019-11-12) - judged not relevant
- P13: [WAL compression setting after PostgreSQL LZ4 default change](https://commitfest.postgresql.org/patch/6979/) (cf, Withdrawn, 2026-07-03) - judged not relevant
- P14: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P15: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P16: [Opclass parameters](https://commitfest.postgresql.org/patch/1559/) (cf, Returned with feedback, 2018-03-01) - judged not relevant
- P17: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1060/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P18: [BRIN Sort - sorting using BRIN indexes](https://commitfest.postgresql.org/patch/3949/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P19: [Opclass parameters](https://commitfest.postgresql.org/patch/2183/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P20: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant
- P21: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P22: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1276/) (cf, Returned with feedback, 2018-02-01) - judged not relevant
- P23: [REINDEX backend filtering](https://commitfest.postgresql.org/patch/2895/) (cf, Withdrawn, 2021-05-08) - judged not relevant
- P24: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 85 s; cost reported by the CLI: $1.12
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 56 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:53:03.901Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
