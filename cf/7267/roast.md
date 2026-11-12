# Design pre-review: Direct TOAST v2, faster, smaller and no migration needed

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7267](https://commitfest.postgresql.org/patch/7267/); [thread](https://www.postgresql.org/message-id/flat/CAMT0RQT5HHYLGWOWPh%3DsceKS7%2BjEE5jTULRTWC9aY9vYjF8OvA%40mail.gmail.com), 31 of 31 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is stated and the claimed gains are large, but the design does not yet have reviewer agreement. A direct pointer carries no value identity, so a reused TID can return another value's bytes without an error (replication-ondisk-1). Storing a TID in the parent removes rewriting the TOAST table alone as a way to remove bloat (replication-ondisk-2). Both follow from the core choice, and the thread shows no accepted answer. The demand case (demand-1) rests on benchmarks without configuration or scripts.

Not checked: Without the code, three points could not be checked: whether the hio.c and pruneheap.c logic runs only for direct TOAST tables, which lock the in-place upgrade takes, and whether prune-to-LP_UNUSED is safe next to indexed plain chunks.

## Prior attempts

- [Support for 8-byte TOAST values (aka the TOAST infinite loop problem)](https://www.postgresql.org/message-id/flat/aFOnKHG7Wn-Srnpv@paquier.xyz) (related; Returned with feedback, 2026-02-10). "Anyway, as far as things stand today on this thread, I have not been able to get even a single +1 for this design." ([message](https://www.postgresql.org/message-id/aTn4aS32O4lmaPnj%40paquier.xyz)) Different now: P7 widened the value ID to 8 bytes and added a callback layer per vartag. Direct TOAST drops the value ID and the index lookup, and addresses chunks by TID. Only oid8 from P7 was committed, and Direct TOAST builds on it.

## Objections a reviewer is likely to raise

### 1. Stale direct pointers can read another value's chunk without error

A DDL command scans with SnapshotAny and detoasts a recently dead tuple, as described in message #28. Its direct chunks were already pruned to LP_UNUSED, and a new insert reused the line pointer. Today this case raises a missing-chunk error, but with direct TOAST the backend returns another row's bytes and nobody notices.

Plain TOAST checks chunk_id and chunk_seq on every fetch, and messages #2, #21 and #28 describe real bugs that this check catches. A direct pointer carries only a TID, and the chunk carries no link to its value. Patch 0009 adds prune-to-LP_UNUSED, so line pointers get reused quickly and a stale TID is more likely to land on a live chunk. Message #25 offers to look into chunk numbers and a back-pointer, but v15 does not include either.

**Question:** What data in the direct pointer or the chunk lets a fetch detect a reused or wrong TID and raise an error instead of returning foreign data?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#2](https://www.postgresql.org/message-id/ap4r6H0A-5SD8pGj%40paquier.xyz), [#21](https://www.postgresql.org/message-id/CAEze2WjaraCS9Ux9Q8aX38WaqHjvF5HnBM%2BoAPLiHUaZ7wjVpQ%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/arx6V8dC3EhZrDBq%40paquier.xyz), [#25](https://www.postgresql.org/message-id/CAMT0RQToBWs4h2S2FzdfES%2BGOYgHRWbeG%3DAxtnbRd6jDYJHHPQ%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAEze2WjEdPEaReJQNPu6%2BNzbFBW9LBazJ6Ls%3DufeHk2v%2BsNqyw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Physical TIDs remove TOAST-only rewrite as a way to remove bloat

A table with compact HOT-updated rows and churning jsonb columns builds up a bloated direct TOAST table. The DBA runs VACUUM FULL or REPACK on the TOAST table and gets an error. The only remaining option is to rewrite the main table and rebuild all its GIN, trigram or HNSW indexes.

The chunk location is stored in the parent tuple, so any move of a chunk requires rewriting the parent. Messages #21, #22 and #28 say that rewriting only the TOAST table is a feature users rely on. Message #28 asks how to shrink a TOAST table whose surviving chunks sit on its last pages, and message #29 answers with a per-row update script and a planned REPACK TOAST. That plan would still have to rewrite parent tuples.

**Question:** How does a planned REPACK TOAST move direct chunks without rewriting parent tuples, and what lock does it hold on the main table while it does so?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#2](https://www.postgresql.org/message-id/ap4r6H0A-5SD8pGj%40paquier.xyz), [#21](https://www.postgresql.org/message-id/CAEze2WjaraCS9Ux9Q8aX38WaqHjvF5HnBM%2BoAPLiHUaZ7wjVpQ%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/arx6V8dC3EhZrDBq%40paquier.xyz), [#24](https://www.postgresql.org/message-id/CAMT0RQQeDnxX%2Bzk94rcNdew7%2B_oM37Wsegyh4mTHqTJ%3D1YGVYQ%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAEze2WjEdPEaReJQNPu6%2BNzbFBW9LBazJ6Ls%3DufeHk2v%2BsNqyw%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/CAMT0RQQhiFfhZGG%3DUdL-MS4QJWK0Ub-dCP7y8DYe_y6ENTkT6w%40mail.gmail.com); `replication-ondisk-2`</sub>

### 3. A bespoke chunk tree duplicates B-tree work outside the index AM

Committers improve B-tree storage, deduplication or vacuum, and TOAST does not benefit from it. The tid[] tree with fanout 50, int8 offsets, recursive delete, amcheck checks and reorderbuffer rebuild code then needs its own fixes for corruption, recovery and decoding bugs, and few people know that code.

Message #1 says the B-tree inside the TOAST table added more code than it removed. Message #28 describes direct TOAST as a specialized inline index and suggests the same effort could go into making the index path faster for TOAST. The read gain is not settled: message #8 says plain matched direct on cached 1 GB reads. Message #21 estimates about 5x fewer page reads, not 10x. No committer in the thread has offered to own the new structure.

**Question:** Which of the measured gains cannot be reached by a narrower B-tree or TOAST index optimization that keeps chunk identity and standalone REPACK?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAMT0RQT5HHYLGWOWPh%3DsceKS7%2BjEE5jTULRTWC9aY9vYjF8OvA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAMT0RQR2Fh8DQsOL%3DMv90CBv856pDry5TQTsT3KfEZ%3DY9%3DbJPA%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAEze2WjaraCS9Ux9Q8aX38WaqHjvF5HnBM%2BoAPLiHUaZ7wjVpQ%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/arx6V8dC3EhZrDBq%40paquier.xyz), [#28](https://www.postgresql.org/message-id/CAEze2WjEdPEaReJQNPu6%2BNzbFBW9LBazJ6Ls%3DufeHk2v%2BsNqyw%40mail.gmail.com); `maintenance-fit-3`</sub>

### 4. Two pointer formats in one table add a branch to every TOAST consumer

A future patch to heap rewrite, logical decoding or an extension handles VARTAG_ONDISK pointers and does not handle VARTAG_DIRECT. A direct pointer then reaches code that reads va_tid bytes as a value ID, and nothing fails until a user hits that path on real data.

Plain and direct pointers coexist in one table with no time limit, so every caller of VARATT_IS_EXTERNAL_ONDISK must handle both formats. Message #1 says the checks were placed next to the existing ones. v15 (message #31) fixed exactly this kind of bug: toast_preserve_valueid read va_tid as an OID after a flavour change. The proposal does not say what would make a missed branch fail loudly in cassert builds or in tests.

**Question:** What mechanism makes code that handles only the plain on-disk pointer fail visibly when it receives a direct pointer?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAMT0RQT5HHYLGWOWPh%3DsceKS7%2BjEE5jTULRTWC9aY9vYjF8OvA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/ap4r6H0A-5SD8pGj%40paquier.xyz), [#31](https://www.postgresql.org/message-id/CAMT0RQSSSq8-5DL6w3f%2Bdh1R9bAyGmega41OcDoQt_rKbMMFGQ%40mail.gmail.com); `maintenance-fit-2`</sub>

### 5. Remaining demand rests on author-run benchmarks after OID8 TOAST was committed

A committer pushes Direct TOAST mainly for the out-of-OIDs case, which OID8 TOAST already handles. Users of tables with heavy TOAST update churn turn it on for speed. The speedup turns out smaller than the thread claimed, and the project keeps a second TOAST format that few people use.

The only field case, a table that ran out of TOAST OIDs at 1 billion rows (message #3), is now covered by OID8 TOAST, which v6 rebased onto (message #11). What remains is performance, measured by the author alone. Message #2 notes the main benchmark has no configuration or scripts, and message #8 retracts the 2x cold-read result. No other user reports TOAST index overhead as a production problem, and message #4 is interest from one prior implementer.

**Question:** Which production workloads already on OID8 TOAST show TOAST index cost as a bottleneck, measured with published, reproducible scripts?

<sub>fixable, level 1b, demand; messages [#2](https://www.postgresql.org/message-id/ap4r6H0A-5SD8pGj%40paquier.xyz), [#3](https://www.postgresql.org/message-id/CAMT0RQQ9E74sOfs_7VdvFEdjSFFW6mOztt2PpmPziNWC6_z6%2Bw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAN-LCVM-rh9j_YbG93S7RUBbeu6-1TEWe4FJQrPqUvmR2cM2JQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAMT0RQR2Fh8DQsOL%3DMv90CBv856pDry5TQTsT3KfEZ%3DY9%3DbJPA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CAMT0RQQvk50Gz-5o%3DvU7oqmK1gXzzVKiMJiwrkUczETTPibAyA%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. What data in the direct pointer or the chunk lets a fetch detect a reused or wrong TID and raise an error, including for single-chunk values?
2. How does a user remove bloat from a direct TOAST table with core commands, without rewriting the main table and rebuilding its indexes?
3. With oid8 TOAST as the baseline, which gains remain under published configuration and scripts, and which of them could a TOAST-specific B-tree improvement not reach?
4. What does pg_dump --binary-upgrade record so that a 5-column TOAST table with a partial index survives pg_upgrade when toast_flavour is unset or 'plain'?
5. What lock does the in-place change of pg_index and pg_constraint take, and what happens to concurrent writers on the table during it?

## Touch points

- **on-disk format / pg_upgrade:** New on-disk pointer tag VARTAG_DIRECT and a 5-column TOAST table format with a partial index.
- **WAL / replication / standby:** Patch 0004 changes decode.c, reorderbuffer.c, proto.c and pgoutput.c to reassemble direct chunks by TID.
- **catalog changes:** TOAST tables gain two columns, the primary key becomes a partial unique index in place, and pg_proc gains a function.
- **ABI / extension API:** varatt.h adds a vartag that extensions and output plugins testing VARATT_IS_EXTERNAL_ONDISK do not know.
- **new GUC:** Adds toast_default_flavour and direct_toast_self_prune, plus the toast_flavour reloption.
- **pg_dump / restore round-trip:** pg_dump --binary-upgrade must recreate a 5-column TOAST table even when the reloption or GUC says 'plain'.
- **security / privileges:** Adds superuser function pg_ensure_direct_toast; a reused TID could return bytes of another column or row.
- **maintenance burden ("who maintains this in 10 years"):** A second TOAST format with its own chunk tree, delete, decoding and amcheck code stays as long as data uses it.
- **performance on the hot path:** Changes detoast.c for all TOAST reads, and message #31 lists changes in hio.c and pruneheap.c.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** oid8 TOAST already covers OID exhaustion, and message #28 suggests improving the TOAST index path instead.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier, author of the 8-byte TOAST work (P7), takes part but says he cannot support the design. An owner would need to know heap pruning, TOAST and logical decoding.
- Priority: needs design rethink; audience niche; review cost high. The target is large tables with many toasted columns and heavy update churn. The series adds an on-disk format across TOAST, heap pruning, logical decoding and amcheck, and reviewers reject the loss of chunk identity and of standalone TOAST repack.

---

[roast-full.md](roast-full.md) has 5 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 10.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
