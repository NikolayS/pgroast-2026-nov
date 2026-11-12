# Design pre-review: BUG #19686: Rolling back SET TABLESPACE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7312](https://commitfest.postgresql.org/patch/7312/) (Needs review); authors: Manuel Reyes Bravo (manu15)
- Thread: [BUG #19686: Rolling back SET TABLESPACE](https://www.postgresql.org/message-id/flat/CAE8JnxP%2BUbdj-kaxfgNt%3D3_FzD3GgOc3gEvPAvrXKfgTgH2q5w%40mail.gmail.com), started 2026-09-16
- Messages read: 33 of 33
- Patch prose read: v4-copy-indices-on-alter-table-set-tablespace.patch, v1-0001-nbtree-skip-insertion-of-existing-tuples.patch, v1-0002-logging-and-testcase.patch, v1-0004-establish-expected-tablespace.out.patch, v1-0003-fix-deferred-relation-copy.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real and committers in the thread back the eager index copy, so levels 1a and 1b pass. Two fixable design points remain at level 2, both about the shortcut and the back branches: standards-compat-1 (space needs depend on the protocol message type) and replication-ondisk-1 (an uninitialized isTopLevel from an extension caller can skip the copy). No committer has replied to v6 or v7, so the choice between them is still open.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it could not be checked whether any non-plain ALTER path or extension caller still reaches the shortcut, or whether WAL replay of the index copies is correct. The pinned-buffer tests for the deferred copy cover only btree and hash in one backend.

## The proposal, restated

**Problem.** ALTER TABLE ... SET TABLESPACE gives the heap a new relfilenode but leaves indexes on their old files. If the transaction then writes to the table and aborts, the heap changes vanish but the index entries survive, so later inserts reuse freed TIDs. This gives assertion failures in nbtree, duplicate rows in gist and wrong query results.

**Proposed design.** The current patch (v7) gives each index a new relfilenumber, copied within the index's own tablespace, when the table is moved, so heap and index files roll back together. The copy is skipped for a plain single-table SET TABLESPACE sent alone as a simple Query outside a transaction block with no ddl_command_end trigger, and for index files created in the current subtransaction.

**User-visible surface.**
- ALTER TABLE/ALTER MATERIALIZED VIEW ... SET TABLESPACE copies the table's indexes in some cases, so it needs free space in the indexes' own tablespace.
- More WAL and one AccessExclusiveLock per index when the indexes are copied.
- Back branches: a new field at the end of AlterTableUtilityContext, which extensions such as TimescaleDB and Apache AGE build.
- New backend accessor IsExtendedQueryMessage() in postgres.c.

**What the author claims.**
- v7 passes check-world, a randomized savepoint test of 40,000 transactions, standby, crash and wal_level=minimal checks on master and REL_14 to REL_19.
- The index copy made the ALTER at most about 1.7 times slower in the author's tests.
- v7 keeps master's behavior for batches in pgJDBC, pgx and psycopg; v6 left the move committed after a later failure.
- Extensions built against unpatched headers run unchanged on patched back branches.
- The deferred-copy design passed 13 pinned-buffer scenarios, which the author offers as a possible v8 for 19 and master.

**Thread state.** Tom Lane, Andres Freund and Michael Paquier rejected copying at commit and backed eager index copy with Andres's two shortcuts; Shihao Zhong reviewed v5 to v7. Open: v6 versus v7, the space cost on a full index tablespace, and whether a deferred copy is worth another look for 19.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-16; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-branch ABI change lets an uninitialized field skip the copy

fixable · level 2, rubric item 11 · ABI / extension API, backpatchability, on-disk format / pg_upgrade · `replication-ondisk-1`

An extension built against unpatched minor-release headers fills AlterTableUtilityContext on the stack and calls AlterTable() for SET TABLESPACE. The new trailing isTopLevel field holds uninitialized stack memory, so the shortcut may be taken where it should not be. A later write and abort in that statement leave stale index entries, found later as wrong results or nbtree assertion failures.

Message #28 says an uninitialized value "can at worst skip the copy, as today", but today's behavior is the corruption the patch fixes. So an old-header build silently keeps the bug on a patched server, and nothing reports it. The material does not say which other conditions, such as the simple-Query and transaction-block checks, still guard this path when the caller is an extension. The author already offers to always copy in back branches (message #28), which avoids the struct change.

**Precedents.**
- [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (Partially reverted, 2022-01-03); thread: [#1](https://www.postgresql.org/message-id/flat/3179992.1641150853@sss.pgh.pa.us), [#2](https://www.postgresql.org/message-id/flat/17350-b5bdcf476e5badbb@postgresql.org)

Thread: [#28 (Manu, 2026-10-04)](https://www.postgresql.org/message-id/179110230672.1690838.3721224931432361508%40gmail.com), [#31 (Manu, 2026-10-14)](https://www.postgresql.org/message-id/179195789433.990372.6387431139267045114%40gmail.com)

**Question:** With an uninitialized isTopLevel from an extension caller, which independent check still forces the index copy, or will back branches always copy instead?

Counter-check: held. Message #28 accepts that an uninitialized isTopLevel can skip the copy, which leaves the bug in place for that caller without any error. P24 must not be used as support, because it records a minor-release struct change that the project accepted.

### 2. Space needs of SET TABLESPACE depend on the client protocol

fixable · level 2, rubric item 12 · backpatchability, wire protocol, SQL-standard conflict or non-standard syntax · `standards-compat-1`

After a minor update, an application sends a plain `alter table t set tablespace ts` through pgJDBC or pgx while the volume holding the indexes is nearly full. The ALTER now fails with "could not extend file", but the same statement typed in psql succeeds. The DBA finds this at the next routine move, on a back branch.

With v7, whether the indexes are copied depends on the wire message type (simple Query or extended protocol), not only on what the SQL says. Message #32 shows a single extended-protocol ALTER failing with ENOSPC where master succeeds. Existing top-level checks such as PreventInTransactionBlock use isTopLevel and XACT_FLAGS_NEEDIMMEDIATECOMMIT, and v7 adds a new IsExtendedQueryMessage() accessor in their place. No committer has ruled on v6 versus v7 (messages #29 to #31), and the material has no documentation hunk for the new space rule.

Thread: [#28 (Manu, 2026-10-04)](https://www.postgresql.org/message-id/179110230672.1690838.3721224931432361508%40gmail.com), [#29 (shihao zhong, 2026-10-04)](https://www.postgresql.org/message-id/CAGRkXqSwuvuYrKdf5o9Jn%3DJqkdO7YT0%2BnTS6L9tf_%2BHu-_gWaQ%40mail.gmail.com), [#30 (Manu, 2026-10-04)](https://www.postgresql.org/message-id/179114425825.497956.17396695057926688547%40gmail.com), [#31 (Manu, 2026-10-14)](https://www.postgresql.org/message-id/179195789433.990372.6387431139267045114%40gmail.com), [#32 (shihao zhong, 2026-10-18)](https://www.postgresql.org/message-id/CAGRkXqRKCNG1LKBV7d98QSdZKf1iT8Qb7LM9jTDKofn33hdsyw%40mail.gmail.com), [#33 (Manu, 2026-10-18)](https://www.postgresql.org/message-id/179234193175.237803.1636724777877426621%40gmail.com)

**Question:** Why should a minor release make the same SET TABLESPACE need extra space in the index tablespace only for extended-protocol clients, and where will users read about it?

Counter-check: held. Message #32 shows a single extended-protocol ALTER failing with ENOSPC where master succeeds, and no committer has chosen between v6 and v7. One correction: v7 keeps the isTopLevel test and adds IsExtendedQueryMessage() to it, so the accessor does not replace it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Each index gets a new relfilenumber and a copied file when its table moves. |
| WAL / replication / standby | **yes** | Each index copy writes WAL of about the index size, which standbys replay. |
| wire protocol | **yes** | v7 skips the index copy only for a lone simple Query, using the new IsExtendedQueryMessage(). |
| catalog changes | no |  |
| ABI / extension API | **yes** | Back branches add a field at the end of AlterTableUtilityContext, which extensions build. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Space needs of SET TABLESPACE depend on the client protocol |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The fix targets REL_14 to REL_19, with new space needs and a struct change in minor releases. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The shortcut conditions in tablecmds.c depend on protocol state and event trigger checks. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. With an uninitialized isTopLevel from an extension built against unpatched headers, which independent check still forces the index copy?
2. Should back branches always copy the indexes and leave AlterTableUtilityContext unchanged, as message #28 offers?
3. Is it acceptable in a minor release that a single extended-protocol SET TABLESPACE fails with ENOSPC where master succeeds (message #32)?
4. Where will users read that SET TABLESPACE can need free space of about the index size in the indexes' own tablespace, and when the shortcut applies?
5. Which version is proposed for commit: v6, v7, or a deferred-copy v8 for 19 and master?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane, Andres Freund and Michael Paquier set the eager-copy direction in the thread, but none has replied to v6, v7 or the deferred-copy question. The owner needs to know tablecmds.c, relation storage and back-branch ABI rules.
- **Liability:** A back-patched corruption fix in REL_14 to REL_19: new space and WAL cost for SET TABLESPACE, a struct change extensions see, and shortcut conditions that must stay correct as protocol and trigger code change.
- **Author follow-up:** Good sign: the author produced seven versions, tested back branches, drivers and extensions, and offers a v8.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** This fixes reproducible index corruption and wrong results (message #1), and committers already agreed on the eager copy. The patch touches mainly tablecmds.c, but the back-branch and shortcut choices need a committer's ruling.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-2` Minor release makes SET TABLESPACE fail on a full index volume. Refuted (not specific): Duplicate of standards-compat-1, with the same single extended-protocol ALTER scenario. Tom Lane, Andres Freund and Michael Paquier already weighed the general space and WAL cost of the eager copy in messages #9, #10, #21 and #23.
- `maintenance-fit-1` The index-copy shortcut depends on listing every later writer. Refuted (already handled): Message #26 stops listing writers: v6 allows the shortcut only for a plain single-table SET TABLESPACE, and v7 narrows it to a lone simple Query. The remaining event trigger case is superuser-only, which Andres accepted in message #21.
- `maintenance-fit-2` Back branches read a new struct field extensions never set. Refuted (not specific): Duplicate of replication-ondisk-1, which also notes that the material does not say whether the transaction-block and simple-Query checks still guard an extension caller.
- `maintenance-fit-3` DDL cost and space needs now depend on the wire protocol. Refuted (not specific): Duplicate of standards-compat-1, which covers the same protocol-dependent ENOSPC failure. The claim that no other code path depends on protocol state is not shown in the material.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent matches: [P24] and [P11] share keywords but died for reasons that do not apply here. Inside the thread, the deferred copy was dropped after messages #7, #9 and #22; v7 follows the eager-copy consensus of messages #9, #10, #21 and #23, and message #33 reopens the deferred copy only as a possible v8, not as the current proposal.
- standards-compat: 1 objection(s). The main design, an eager index copy, has committer backing (messages #9, #10, #21, #23). The objection above covers only how the shortcut behaves for users.
- replication-ondisk: 2 objection(s). The nbtree VACUUM invariant does not apply: v7 does not touch nbtree, and the v1 nbtree change was dropped. The author reports standby, crash and wal_level=minimal checks as clean.
- security: 0 objection(s). No security objection. The index copy goes into the index's existing tablespace, where REINDEX and ordinary inserts already write without a new CREATE check, and the only owner-reachable case that bypassed the shortcut (CHECK on an inheritance child, message #26) was closed in v6. Not checked without code: whether any non-plain ALTER path still reaches the shortcut.
- maintenance-fit: 3 objection(s). No hot-path cost for non-users is visible. The nbtree VACUUM invariant does not apply, since the nbtree patch (v1-0001) was dropped in message #4.
- already-exists: 0 objection(s). This proposal fixes data corruption, and no existing feature, setting or extension prevents it. Running SET TABLESPACE outside a transaction block avoids the bug in practice, but core does not enforce that, so it is only a workaround. The shortcut reuses the existing isTopLevel and transaction-block checks, which the thread discussed and settled in messages #21 to #30.
- demand: 0 objection(s). The proposal fixes a reported data-corruption bug (BUG #19686), so the user problem is real and reproducible: message #1 shows wrong query results, and messages #16 and #24 show stale index entries on master. Tom Lane, Andres Freund and Michael Paquier agree the bug must be fixed and support eager index copy (messages #7, #9, #10, #23). The shortcuts that add complexity were requested by Andres and backed by Tom, so they are not unrequested features. The one open demand-type question is whether a deferred copy for 19 is worth another look (message #33). That is only an offer, not part of v7, so it is not an objection here.

### Precedents retrieved

- P1: [Revert change to ALTER TABLESPACE summary.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=155c0f24b7d370eb57fb8dde0447e11e41c405c2) (revert, Reverted, 2014-11-14) - judged not relevant
- P2: [Allow TOAST tables to be moved to a different tablespace](https://commitfest.postgresql.org/patch/16/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P3: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant; reason summarised from its thread
- P4: [default_index_tablespace](https://commitfest.postgresql.org/patch/216/) (cf, Returned with feedback, 2015-06-14) - judged not relevant; reason summarised from its thread
- P5: [Allow TOAST tables to be moved to a different tablespace](https://commitfest.postgresql.org/patch/188/) (cf, Returned with feedback, 2015-08-12) - judged not relevant; reason summarised from its thread
- P6: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P7: [Move tablespace path re-creation from the makefiles to pg_regress (test for mkdir function in the backend)](https://commitfest.postgresql.org/patch/3067/) (cf, Withdrawn, 2021-04-16) - judged not relevant; reason summarised from its thread
- P8: [Allow pg_upgrade to copy segments of the same relfilenode in parallel](https://commitfest.postgresql.org/patch/3525/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P9: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant; reason summarised from its thread
- P10: [making relfilenodes 56 bit](https://commitfest.postgresql.org/patch/3711/) (cf, Withdrawn, 2023-01-06) - judged not relevant
- P11: [Revert "Skip WAL for new relfilenodes, under wal_level=minimal."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=de9396326edcbe5cafc06a72016f9d715c350e0e) (revert, Reverted, 2020-03-22) - used in the review
- P12: [Inconsistency in owner assignment between INDEX and STATISTICS](https://commitfest.postgresql.org/patch/6571/) (cf, Withdrawn, 2026-03-11) - judged not relevant
- P13: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P14: [Revert indexed and enlargable binary heap implementation.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=810f64a01567610af7b92b0de930f16f3e20064e) (revert, Reverted, 2024-04-11) - judged not relevant
- P15: [Allow per-tablespace quotas](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Tablespaces) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P16: [Reduce lock level for ALTER TABLE ... ADD CHECK](https://commitfest.postgresql.org/patch/3088/) (cf, Withdrawn, 2021-11-21) - judged not relevant
- P17: [CONTEXT of ALTER TABLE behind-the-scenes effects](https://commitfest.postgresql.org/patch/427/) (cf, Returned with feedback, 2016-03-15) - judged not relevant
- P18: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P19: [Enhance file_fdw to report processed and skipped tuples in COPY progress](https://commitfest.postgresql.org/patch/5282/) (cf, Withdrawn, 2025-03-09) - judged not relevant
- P20: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - judged not relevant
- P21: [Reset vacuum_defer_cleanup_age to PGC_SIGHUP. Revert commit 84725aa5efe11688633b553e58113efce4181f2e](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f480e294498533820f3ef3e6de4dcb8ff5401140) (revert, Partially reverted, 2013-02-04) - judged not relevant
- P22: [Add "\gi file" to psql to allow COPY with variable expansion](https://commitfest.postgresql.org/patch/5674/) (cf, Returned with feedback, 2025-06-17) - judged not relevant
- P23: [COPY result file's umask](https://commitfest.postgresql.org/patch/383/) (cf, Rejected, 2015-11-17) - judged not relevant
- P24: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - used in the review

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 136 s; cost reported by the CLI: $2.77
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 77 text field(s) checked, 7 finding(s), 7 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:39:51.390Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
