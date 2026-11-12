# Design pre-review: Logical replication row filter loses unchanged toasted columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7188](https://commitfest.postgresql.org/patch/7188/); [thread](https://www.postgresql.org/message-id/flat/CAOzEurQaGBrDu8hFw%2Bp16_uf_%2Bb94A276fYgzwUOK5X-bSX%2Bjg%40mail.gmail.com), 41 of 41 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The proposal fixes reproducible silent data loss in a released feature, and no surviving objection disputes the goal or the logging approach on master. The surviving objections are fixable design points, and most concern back-patching: the wider PANIC path (replication-ondisk-1), the PublicationDesc change (maintenance-fit-2) and the changed old-key tuple content (standards-compat-1). A committer who knows heap WAL logging and logical decoding has to decide those. Confidence is medium because the material shows no patch prose for v3 to v6.

Not checked: The material has no patch prose or code for v3 to v6. The exact content of the old tuple, any size check before the critical section and the relcache invalidation of rf_exists_for_update could not be judged.

## Prior attempts

None found before 2026-08-12.

## Objections a reviewer is likely to raise

### 1. New path extends the XLogRecordMaxSize PANIC to tables without REPLICA IDENTITY FULL

A table with the default replica identity is in a row-filtered publication. A user changes the key of a row whose unchanged toasted columns total about 1021 MB. The flattened old tuple fits MaxAllocSize but exceeds XLogRecordMaxSize, XLogInsert() fails inside the critical section, and the whole server restarts.

Message #36 shows that this size window PANICs today with REPLICA IDENTITY FULL, a setting the user chose. v6 sends the same flattening path to tables whose owners never chose FULL, only because some publication has a row filter. The PANIC fix moved to a separate HEAD-only thread (messages #35, #36), so v6 widens the exposure, and on back branches it would do so with no fix in sight.

**Question:** Does v6 check the old-key size against XLogRecordMaxSize before the critical section, so this case raises an ERROR instead of a PANIC?

<sub>fixable, level 2, WAL / replication / standby; messages [#33](https://www.postgresql.org/message-id/CAA4eK1%2BTx4D%2BkN15AH2%3DvHVGNtdtU%2BP7oFuvk4TXEvwRGA0xbw%40mail.gmail.com), [#35](https://www.postgresql.org/message-id/CAD21AoDEunP%2BdNNRYeKuE4RR5nzw0P43Mceybu4zDrxZoNxDtQ%40mail.gmail.com), [#36](https://www.postgresql.org/message-id/CAA4eK1J1mBaAgnmycYT3-fk%3D8q2E6QOb__Df5yRtA-YLSN1pJA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Back-patching changes an exposed struct and a function name

An extension built against 15.x declares PublicationDesc on the stack and calls RelationBuildPublicationDesc(). After a minor upgrade, the new backend writes rf_exists_for_update past the end of the smaller struct and corrupts the extension's stack. Users find out from crashes or wrong values after a routine update.

Amit described this exact out-of-bounds write in message #14. Placing the field last (message #24) keeps the existing offsets but does not prevent that write. The only evidence of safety is a GitHub and Debian code search (message #18), which cannot see private extensions. v6 also renames ExtractReplicaIdentity() to BuildOldKeyTuple(), and no branch list or back-branch variant of v3 to v6 is in the material.

**Question:** Which branches get v6, and does the back-branch version leave PublicationDesc and existing function names untouched?

<sub>fixable, level 2, backpatchability; messages [#14](https://www.postgresql.org/message-id/CAA4eK1%2BEFf7k2cKuzhLd6mq%3DFb2qRKR6YZ59k0cytYwwU5z1vA%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/TY4PR01MB17718D001102126DA402F622994A72%40TY4PR01MB17718.jpnprd01.prod.outlook.com), [#24](https://www.postgresql.org/message-id/CAA4eK1LMLnZsZLEoHFDQODqUzECj-UUm74nMt0tKDZCJ0pWy3g%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Back-patched old-key tuples change what existing decoders emit

A table is in a row-filtered publication and is also decoded by test_decoding, wal2json or a CDC tool that reads pgoutput. After a minor upgrade, an UPDATE that changes the key logs the unchanged large column in the old tuple. Those consumers start emitting a full multi-MiB value in their 'old key' or 'before' image, and nothing in the release says why.

Message #25 makes BuildOldKeyTuple() put non-replica-identity columns into the old-key tuple, but pgoutput still flags it 'K'. The claim of no protocol change covers message format only, and the content of a key tuple is part of the contract that consumers depend on. Decoders that do not use publications receive this extra data without gaining anything from it, a side of Matthias's message #10 that the thread did not answer. Nobody in the thread checked what test_decoding or pgoutput's 'K' tuple now carries.

**Question:** What do test_decoding, pgoutput 'K' tuples and third-party plugins emit for the old key after v6, and why is that acceptable in a minor release?

<sub>fixable, level 2, backpatchability; messages [#10](https://www.postgresql.org/message-id/CAEze2WjBem2Lbh%3Dizi8SaPMCqJG5uozMC-rkM0E4HPGEJaogxQ%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAD21AoBE2kS%3DL%3DV7zoWF7Qs39aRSKm%2BzbLoETVRt6mofWZOBzA%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/TY4PR01MB1771882DEFAD86486CB2C700F94A42%40TY4PR01MB17718.jpnprd01.prod.outlook.com); `standards-compat-1`</sub>

### 4. Dropping the subscriber-side check leaves silent NULLs across versions and races

A patched subscriber replicates from an unpatched publisher, or from a patched one where ALTER PUBLICATION ... ADD TABLE ... WHERE commits during the UPDATE (message #34). The transformed INSERT still arrives with the column marked unchanged, and the subscriber stores NULL without any error.

Message #13 argued that the apply-time check is needed on master even with a publisher-side fix, because subscribers receive changes from older publishers. Message #34 shows a race in which v6 still loses the value, and that race moved to a separate thread. The thread does not say whether the check in apply_handle_insert() is still planned.

**Question:** Will the check in apply_handle_insert() go in with v6, so that cases v6 cannot cover fail loudly on the subscriber?

<sub>fixable, level 2, WAL / replication / standby; messages [#6](https://www.postgresql.org/message-id/CA%2BUBoq0jv0BafwSFYDd680SxEwGYUTSjrPW0iQyZdqKFfRZokw%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAOzEurQ41MNd39No%3Dy-JNkz1viNS7SziQ09M_ag7ajZ-KOjw5A%40mail.gmail.com), [#34](https://www.postgresql.org/message-id/CALDaNm2MF18JPUz_rwMzbgeqy1PUwxVKwEU8E1cWPObq-yqxrg%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/CALDaNm2_ViEtbt8c_iPg3wXg773PLX4em-fOEfhkgdUFkjF53A%40mail.gmail.com), [#38](https://www.postgresql.org/message-id/CAA4eK1KQysGW-f_%3D-E_2WFvPVZL_X7E1DY6W_J679qHrtA9e6Q%40mail.gmail.com); `replication-ondisk-4`</sub>

### 5. No measurements of extra WAL in a minor-release change

A site with a row-filtered table of large documents updates keys often, and most rows never enter the filter. After a minor upgrade, each such UPDATE detoasts and logs every unchanged out-of-line value. WAL volume, archive storage and replication lag grow without any configuration change, and the cause is hard to find.

The trigger cannot tell whether the filter result changes between the old and new row, so extra logging happens even when no transformation follows (message #22). Message #3 called this kind of WAL regression a reason not to back-patch option B. Message #14 promised performance tests, but no WAL or CPU numbers appear in the thread. The cost falls only on row-filter users, which lowers the concern for master but not for a minor release.

**Question:** What extra WAL and CPU does v6 add for key-changing updates on a row-filtered table with large out-of-line columns, measured where most updates do not cross the filter?

<sub>fixable, level 2, performance on the hot path; messages [#3](https://www.postgresql.org/message-id/CA%2BUBoq0bj_b1QXeqTGXqdD9Tw1mvCxaxcBs4makOO%3DuLaOJobg%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAA4eK1%2BEFf7k2cKuzhLd6mq%3DFb2qRKR6YZ59k0cytYwwU5z1vA%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CAA4eK1LrrgE5AbHcRPSSK49D-K2jAbc930KzveQH%3Db_1Ls0Fgw%40mail.gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Does v6 check the old-key tuple size against XLogRecordMaxSize before the critical section, so that the new path raises an ERROR instead of a PANIC?
2. Which branches would get v6, and does the back-branch version leave the size of PublicationDesc unchanged?
3. What do test_decoding and pgoutput 'K' tuples emit for the old key after v6, and why is that acceptable in a minor release?
4. Will the check in apply_handle_insert() go in alongside v6, so that an INSERT from an unpatched publisher fails instead of storing NULL?
5. How much extra WAL and CPU does v6 add for key-changing updates on a row-filtered table when most updates do not cross the filter?

## Touch points

- **WAL / replication / standby:** heap_update() logs unchanged out-of-line values outside the replica identity in the old tuple of UPDATE records.
- **wire protocol:** The message format is unchanged, but the old-key tuple that output plugins receive can now carry columns outside the replica identity.
- **ABI / extension API:** PublicationDesc gains the field rf_exists_for_update, and relcache gains RelationHasPubRowFilterForUpdate().
- **security / privileges:** A role with only UPDATE privilege on a row-filtered table can reach the XLogRecordMaxSize PANIC through the new logging path.
- **backpatchability:** The bug affects 15 and later, and the thread argues for back-patching a change to WAL content and to an exposed struct.
- **maintenance burden ("who maintains this in 10 years"):** heap_update() now depends on publication row filter state, and an INCLUDE-style replica identity may later overlap with it.
- **performance on the hot path:** Key-changing UPDATEs on row-filtered tables detoast and log unchanged out-of-line values, even when no transformation follows.

## Who would commit it, and when

- Owner: qualified committer engaged. It needs someone who knows heap WAL logging and logical decoding. Amit and Sawada take part in the thread and agreed on the logging approach.
- Priority: proceed; audience niche; review cost medium. The bug loses data silently, but only for row-filter users whose UPDATE changes the key of a row with out-of-line columns. The fix touches heap_update(), relcache and decoding. No patch prose for v6 was read, so the review cost is an estimate.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
