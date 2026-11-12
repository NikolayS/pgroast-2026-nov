# Design pre-review: Logical replication row filter loses unchanged toasted columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7188](https://commitfest.postgresql.org/patch/7188/) (Needs review); authors: Zhijie Hou (houzj)
- Thread: [Logical replication row filter loses unchanged toasted columns](https://www.postgresql.org/message-id/flat/CAOzEurQaGBrDu8hFw%2Bp16_uf_%2Bb94A276fYgzwUOK5X-bSX%2Bjg%40mail.gmail.com), started 2026-08-12
- Messages read: 41 of 41
- Patch prose read: 0001-Refuse-a-logical-replication-INSERT-that-is-m-master.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The proposal fixes reproducible silent data loss in a released feature, and no surviving objection disputes the goal or the logging approach on master. The surviving objections are fixable design points, and most concern back-patching: the wider PANIC path (replication-ondisk-1), the PublicationDesc change (maintenance-fit-2) and the changed old-key tuple content (standards-compat-1). A committer who knows heap WAL logging and logical decoding has to decide those. Confidence is medium because the material shows no patch prose for v3 to v6.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The material has no patch prose or code for v3 to v6. The exact content of the old tuple, any size check before the critical section and the relcache invalidation of rf_exists_for_update could not be judged.

## The proposal, restated

**Problem.** When a publication row filter turns an UPDATE into an INSERT, an unchanged out-of-line column outside the replica identity reaches the subscriber as NULL. On a NOT NULL column the apply worker fails with a constraint violation instead. The bug affects logical replication users on 15 and later who use row filters.

**Proposed design.** The current patch (v6) changes heap_update() to add unchanged out-of-line values outside the replica identity to the logged old tuple. This happens only when a publication has a row filter for UPDATE and the update changes the replica identity key. pgoutput's existing copy loop then finds the values and sends a complete INSERT.

**User-visible surface.**
- UPDATE WAL records for row-filtered tables can carry unchanged out-of-line values outside the replica identity in the old tuple, which increases WAL volume.
- New documented limit: an UPDATE or DELETE fails if the logged old row image exceeds about 1 GB, covering REPLICA IDENTITY FULL and this new case.
- New PublicationDesc field rf_exists_for_update and a new relcache accessor, RelationHasPubRowFilterForUpdate().
- ExtractReplicaIdentity() is renamed BuildOldKeyTuple() and also collects unchanged out-of-line columns.

**What the author claims.**
- The bug exists since row filters were added in commit 52e4f0cd472 and affects 15 and later.
- Extra WAL is written only in a narrow case: row filter present, key changed, and old tuple holds external values.
- The fix needs no change to the output plugin, the protocol or the subscriber.
- The fix adds no WAL record type and keeps the WAL format, so Sawada argues it can be back-patched.
- Hou found no extensions that depend on the size of PublicationDesc, so he argues the ABI change is safe to back-patch.

**Thread state.** Amit, Sawada, Kuroda, Shinya and vignesh agreed on the logging approach and moved the 1 GB limit, the concurrent ALTER PUBLICATION race and the XLogRecordMaxSize PANIC to separate threads. The last message (2026-09-08) proposes new documentation wording, and v6 has not been revised since.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-12; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. New path extends the XLogRecordMaxSize PANIC to tables without REPLICA IDENTITY FULL

fixable · level 2, rubric item 7 · WAL / replication / standby, backpatchability · `replication-ondisk-1`

A table with the default replica identity is in a row-filtered publication. A user changes the key of a row whose unchanged toasted columns total about 1021 MB. The flattened old tuple fits MaxAllocSize but exceeds XLogRecordMaxSize, XLogInsert() fails inside the critical section, and the whole server restarts.

Message #36 shows that this size window PANICs today with REPLICA IDENTITY FULL, a setting the user chose. v6 sends the same flattening path to tables whose owners never chose FULL, only because some publication has a row filter. The PANIC fix moved to a separate HEAD-only thread (messages #35, #36), so v6 widens the exposure, and on back branches it would do so with no fix in sight.

Thread: [#33 (Amit Kapila, 2026-09-01)](https://www.postgresql.org/message-id/CAA4eK1%2BTx4D%2BkN15AH2%3DvHVGNtdtU%2BP7oFuvk4TXEvwRGA0xbw%40mail.gmail.com), [#35 (Masahiko Sawada, 2026-09-02)](https://www.postgresql.org/message-id/CAD21AoDEunP%2BdNNRYeKuE4RR5nzw0P43Mceybu4zDrxZoNxDtQ%40mail.gmail.com), [#36 (Amit Kapila, 2026-09-02)](https://www.postgresql.org/message-id/CAA4eK1J1mBaAgnmycYT3-fk%3D8q2E6QOb__Df5yRtA-YLSN1pJA%40mail.gmail.com)

**Question:** Does v6 check the old-key size against XLogRecordMaxSize before the critical section, so this case raises an ERROR instead of a PANIC?

Counter-check: held. Message #36 reproduces the PANIC with REPLICA IDENTITY FULL, and v6 uses the same flattening for row-filtered tables. The thread moved the fix to a separate HEAD-only effort and does not say that v6 checks the size before the critical section.

### 2. Back-patching changes an exposed struct and a function name

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `maintenance-fit-2`

An extension built against 15.x declares PublicationDesc on the stack and calls RelationBuildPublicationDesc(). After a minor upgrade, the new backend writes rf_exists_for_update past the end of the smaller struct and corrupts the extension's stack. Users find out from crashes or wrong values after a routine update.

Amit described this exact out-of-bounds write in message #14. Placing the field last (message #24) keeps the existing offsets but does not prevent that write. The only evidence of safety is a GitHub and Debian code search (message #18), which cannot see private extensions. v6 also renames ExtractReplicaIdentity() to BuildOldKeyTuple(), and no branch list or back-branch variant of v3 to v6 is in the material.

Thread: [#14 (Amit Kapila, 2026-08-17)](https://www.postgresql.org/message-id/CAA4eK1%2BEFf7k2cKuzhLd6mq%3DFb2qRKR6YZ59k0cytYwwU5z1vA%40mail.gmail.com), [#18 ("Zhijie Hou (Fujitsu)", 2026-08-17)](https://www.postgresql.org/message-id/TY4PR01MB17718D001102126DA402F622994A72%40TY4PR01MB17718.jpnprd01.prod.outlook.com), [#24 (Amit Kapila, 2026-08-20)](https://www.postgresql.org/message-id/CAA4eK1LMLnZsZLEoHFDQODqUzECj-UUm74nMt0tKDZCJ0pWy3g%40mail.gmail.com)

**Question:** Which branches get v6, and does the back-branch version leave PublicationDesc and existing function names untouched?

Counter-check: held. Amit described the out-of-bounds stack write in message #14, and a public code search cannot rule it out. The rename must not be used as support, because ExtractReplicaIdentity() is a static function in heapam.c.

### 3. Back-patched old-key tuples change what existing decoders emit

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby, wire protocol · `standards-compat-1`

A table is in a row-filtered publication and is also decoded by test_decoding, wal2json or a CDC tool that reads pgoutput. After a minor upgrade, an UPDATE that changes the key logs the unchanged large column in the old tuple. Those consumers start emitting a full multi-MiB value in their 'old key' or 'before' image, and nothing in the release says why.

Message #25 makes BuildOldKeyTuple() put non-replica-identity columns into the old-key tuple, but pgoutput still flags it 'K'. The claim of no protocol change covers message format only, and the content of a key tuple is part of the contract that consumers depend on. Decoders that do not use publications receive this extra data without gaining anything from it, a side of Matthias's message #10 that the thread did not answer. Nobody in the thread checked what test_decoding or pgoutput's 'K' tuple now carries.

Thread: [#10 (Matthias van de Meent, 2026-08-13)](https://www.postgresql.org/message-id/CAEze2WjBem2Lbh%3Dizi8SaPMCqJG5uozMC-rkM0E4HPGEJaogxQ%40mail.gmail.com), [#21 (Masahiko Sawada, 2026-08-19)](https://www.postgresql.org/message-id/CAD21AoBE2kS%3DL%3DV7zoWF7Qs39aRSKm%2BzbLoETVRt6mofWZOBzA%40mail.gmail.com), [#25 ("Zhijie Hou (Fujitsu)", 2026-08-20)](https://www.postgresql.org/message-id/TY4PR01MB1771882DEFAD86486CB2C700F94A42%40TY4PR01MB17718.jpnprd01.prod.outlook.com)

**Question:** What do test_decoding, pgoutput 'K' tuples and third-party plugins emit for the old key after v6, and why is that acceptable in a minor release?

Counter-check: held. v6 puts non-identity values into the old-key tuple, and every output plugin sees that tuple. The thread does not say what test_decoding or pgoutput then emit, and Matthias's point in message #10 has no answer.

### 4. Dropping the subscriber-side check leaves silent NULLs across versions and races

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-4`

A patched subscriber replicates from an unpatched publisher, or from a patched one where ALTER PUBLICATION ... ADD TABLE ... WHERE commits during the UPDATE (message #34). The transformed INSERT still arrives with the column marked unchanged, and the subscriber stores NULL without any error.

Message #13 argued that the apply-time check is needed on master even with a publisher-side fix, because subscribers receive changes from older publishers. Message #34 shows a race in which v6 still loses the value, and that race moved to a separate thread. The thread does not say whether the check in apply_handle_insert() is still planned.

Thread: [#6 (Nikhil Sontakke, 2026-08-13)](https://www.postgresql.org/message-id/CA%2BUBoq0jv0BafwSFYDd680SxEwGYUTSjrPW0iQyZdqKFfRZokw%40mail.gmail.com), [#13 (Shinya Kato, 2026-08-15)](https://www.postgresql.org/message-id/CAOzEurQ41MNd39No%3Dy-JNkz1viNS7SziQ09M_ag7ajZ-KOjw5A%40mail.gmail.com), [#34 (vignesh C, 2026-09-01)](https://www.postgresql.org/message-id/CALDaNm2MF18JPUz_rwMzbgeqy1PUwxVKwEU8E1cWPObq-yqxrg%40mail.gmail.com), [#37 (vignesh C, 2026-09-02)](https://www.postgresql.org/message-id/CALDaNm2_ViEtbt8c_iPg3wXg773PLX4em-fOEfhkgdUFkjF53A%40mail.gmail.com), [#38 (Amit Kapila, 2026-09-02)](https://www.postgresql.org/message-id/CAA4eK1KQysGW-f_%3D-E_2WFvPVZL_X7E1DY6W_J679qHrtA9e6Q%40mail.gmail.com)

**Question:** Will the check in apply_handle_insert() go in with v6, so that cases v6 cannot cover fail loudly on the subscriber?

Counter-check: held. Message #13 gave the older-publisher case as the reason to keep the apply-time check, and nothing after message #21 says whether it stays. The race in message #34 went to another thread, but the cross-version case does not depend on it.

### 5. No measurements of extra WAL in a minor-release change

fixable · level 2, rubric item 8 · performance on the hot path, WAL / replication / standby, backpatchability · `maintenance-fit-3`

A site with a row-filtered table of large documents updates keys often, and most rows never enter the filter. After a minor upgrade, each such UPDATE detoasts and logs every unchanged out-of-line value. WAL volume, archive storage and replication lag grow without any configuration change, and the cause is hard to find.

The trigger cannot tell whether the filter result changes between the old and new row, so extra logging happens even when no transformation follows (message #22). Message #3 called this kind of WAL regression a reason not to back-patch option B. Message #14 promised performance tests, but no WAL or CPU numbers appear in the thread. The cost falls only on row-filter users, which lowers the concern for master but not for a minor release.

Thread: [#3 (Nikhil Sontakke, 2026-08-12)](https://www.postgresql.org/message-id/CA%2BUBoq0bj_b1QXeqTGXqdD9Tw1mvCxaxcBs4makOO%3DuLaOJobg%40mail.gmail.com), [#14 (Amit Kapila, 2026-08-17)](https://www.postgresql.org/message-id/CAA4eK1%2BEFf7k2cKuzhLd6mq%3DFb2qRKR6YZ59k0cytYwwU5z1vA%40mail.gmail.com), [#22 (Amit Kapila, 2026-08-20)](https://www.postgresql.org/message-id/CAA4eK1LrrgE5AbHcRPSSK49D-K2jAbc930KzveQH%3Db_1Ls0Fgw%40mail.gmail.com)

**Question:** What extra WAL and CPU does v6 add for key-changing updates on a row-filtered table with large out-of-line columns, measured where most updates do not cross the filter?

Counter-check: held. Message #14 promised performance tests and message #22 named the false-positive cost, but the thread has no WAL or CPU numbers. The cost falls only on row-filter users, which lowers the concern for master but does not settle a back-patch.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | heap_update() logs unchanged out-of-line values outside the replica identity in the old tuple of UPDATE records. |
| wire protocol | **yes** | The message format is unchanged, but the old-key tuple that output plugins receive can now carry columns outside the replica identity. |
| catalog changes | no |  |
| ABI / extension API | **yes** | PublicationDesc gains the field rf_exists_for_update, and relcache gains RelationHasPubRowFilterForUpdate(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | A role with only UPDATE privilege on a row-filtered table can reach the XLogRecordMaxSize PANIC through the new logging path. |
| backpatchability | **yes** | The bug affects 15 and later, and the thread argues for back-patching a change to WAL content and to an exposed struct. |
| maintenance burden ("who maintains this in 10 years") | **yes** | heap_update() now depends on publication row filter state, and an INCLUDE-style replica identity may later overlap with it. |
| performance on the hot path | **yes** | Key-changing UPDATEs on row-filtered tables detoast and log unchanged out-of-line values, even when no transformation follows. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v6 check the old-key tuple size against XLogRecordMaxSize before the critical section, so that the new path raises an ERROR instead of a PANIC?
2. Which branches would get v6, and does the back-branch version leave the size of PublicationDesc unchanged?
3. What do test_decoding and pgoutput 'K' tuples emit for the old key after v6, and why is that acceptable in a minor release?
4. Will the check in apply_handle_insert() go in alongside v6, so that an INSERT from an unpatched publisher fails instead of storing NULL?
5. How much extra WAL and CPU does v6 add for key-changing updates on a row-filtered table when most updates do not cross the filter?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. It needs someone who knows heap WAL logging and logical decoding. Amit and Sawada take part in the thread and agreed on the logging approach.
- **Liability:** The owner signs for a change to heap_update() WAL logging on every branch that gets it, including the wider PANIC path and the PublicationDesc change. That lasts for the life of those branches.
- **Author follow-up:** The patch reached v6 within a month of the report, which suggests active follow-up. No new version has appeared since the documentation proposal of 2026-09-08.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The bug loses data silently, but only for row-filter users whose UPDATE changes the key of a row with out-of-line columns. The fix touches heap_update(), relcache and decoding. No patch prose for v6 was read, so the review cost is an estimate.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Minor release makes some UPDATEs fail that worked before. Not listed: not among the strongest objections.
- `maintenance-fit-1` (fixable, level 2) Heap WAL content now depends on publication row filters. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-3` Newer subscribers still store NULL silently from unpatched publishers. Refuted (not specific): Duplicate of replication-ondisk-4, which also covers the concurrent ALTER PUBLICATION case.
- `replication-ondisk-2` Back-patching changes WAL content, WAL volume and failure modes in a minor release. Refuted (not specific): Duplicate of standards-compat-2 (new failure), maintenance-fit-2 (struct change) and maintenance-fit-3 (WAL volume). Messages #3 and #4 are about the wider option B, so they do not support the objection against v6's narrower condition.
- `replication-ondisk-3` Old-key tuple now carries non-identity columns for every logical decoder. Refuted (not specific): Duplicate of standards-compat-1. The row-match failure it adds is not tied to any named plugin.
- `security-1` New logging path widens a PANIC reachable by any user with UPDATE. Refuted (not specific): Duplicate of replication-ondisk-1. The privilege angle adds no new mechanism, because the same size window and the same critical section are involved.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P5] was the original row filter feature, returned only because no new version was posted, and its thread summary lists TOAST handling as an open point. Message #1 cites an earlier discussion on LOGICALREP_COLUMN_UNCHANGED for the fact that unchanged out-of-line values are not in WAL; v6 addresses that fact directly.
- standards-compat: 3 objection(s). Three fixable objections. The thread never shows v3 to v6 patch prose, so the exact contents of the old tuple come from the thread messages alone.
- replication-ondisk: 4 objection(s). Objections are given above.
- security: 1 objection(s). The extra old-tuple data goes only to readers who can already decode full new rows, and pgoutput still applies the filters and column lists, so I see no new data exposure.
- maintenance-fit: 3 objection(s). Three fixable design objections from the maintenance angle. The 1 GB limit, the PANIC path and the ALTER PUBLICATION race were moved to separate threads, so they are not raised here.
- already-exists: 0 objection(s). The only existing alternative is REPLICA IDENTITY FULL. Message #23 shows it cannot be combined with a publication column list and logs the whole old row on every UPDATE, and no extension, contrib module or hook solves this inside the WAL record. The INCLUDE-style replica identity option (messages #5, #7) does not exist yet and cannot be back-patched, so the bug fix does not duplicate it.
- demand: 0 objection(s). The proposal fixes silent data loss in a released feature, row filters on 15 and later, and message #1 gives a reproducible case, so a correctness fix needs no further demand evidence. The extra WAL falls only on tables with a row filter whose update changes the replica identity key, so users of the feature pay for it, and committers and reviewers from several companies are engaged.

### Precedents retrieved

- P1: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P2: [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (cf, Returned with feedback, 2019-09-11) - judged not relevant; reason summarised from its thread
- P3: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1710/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P4: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1862/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P5: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/2270/) (cf, Returned with feedback, 2020-03-16) - used in the review; reason summarised from its thread
- P6: [Revert "Fix bug in checking of IDENTIFY_SYSTEM result."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c4762886539b06da682045e19ba73dcee3789638) (revert, Reverted, 2014-08-19) - judged not relevant
- P7: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P8: [Support for 8-byte TOAST values (aka the TOAST infinite loop problem)](https://commitfest.postgresql.org/patch/5830/) (cf, Returned with feedback, 2026-02-10) - judged not relevant; reason summarised from its thread
- P9: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P10: [64-bit TOAST value ID](https://commitfest.postgresql.org/patch/4296/) (cf, Returned with feedback, 2023-09-01) - judged not relevant; reason summarised from its thread
- P11: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant
- P12: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P13: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P14: [Optimizing NULL check on pkey column](https://commitfest.postgresql.org/patch/2699/) (cf, Returned with feedback, 2021-09-27) - judged not relevant
- P15: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P16: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P17: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant
- P18: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P19: [Change the csv log to 'key:value' to facilitate the user to understanding and processing of log](https://commitfest.postgresql.org/patch/3591/) (cf, Rejected, 2022-04-07) - judged not relevant
- P20: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P21: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant
- P22: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant
- P23: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P24: [Revert the default toast compression strategy to the former behavior where there's no limit on the size of datum we'll try to compress.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=229bffedbc52f12aadb95ca2ee24d5bef36b7c7d) (revert, Reverted, 2009-01-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 168 s; cost reported by the CLI: $3.54
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 104 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:26:21.997Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
