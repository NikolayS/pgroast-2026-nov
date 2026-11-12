# Design pre-review: Direct TOAST v2, faster, smaller and no migration needed

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7267](https://commitfest.postgresql.org/patch/7267/) (Needs review); authors: Hannu Krosing (hannukrosing)
- Thread: [Direct TOAST v2, faster, smaller and no migration needed](https://www.postgresql.org/message-id/flat/CAMT0RQT5HHYLGWOWPh%3DsceKS7%2BjEE5jTULRTWC9aY9vYjF8OvA%40mail.gmail.com), started 2026-09-05
- Messages read: 31 of 31
- Patch prose read: v15-0001-Refactor-detoasting-and-decompression-pipeline-t.patch, v15-0002-Add-Direct-TOAST-catalog-GUC-and-reloptions-infr.patch, v15-0003-Implement-Direct-TOAST-core-storage-reading-and-.patch, v15-0004-Support-Direct-TOAST-in-logical-decoding-replica.patch, v15-0005-Add-amcheck-verification-for-Direct-TOAST-tuples.patch, v15-0006-Add-documentation-for-Direct-TOAST.patch, v15-0007-Add-pg_ensure_direct_toast-for-in-place-legacy-T.patch, v15-0008-Add-backend-TOAST-architecture-documentation-and.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is stated and the claimed gains are large, but the design does not yet have reviewer agreement. A direct pointer carries no value identity, so a reused TID can return another value's bytes without an error (replication-ondisk-1). Storing a TID in the parent removes rewriting the TOAST table alone as a way to remove bloat (replication-ondisk-2). Both follow from the core choice, and the thread shows no accepted answer. The demand case (demand-1) rests on benchmarks without configuration or scripts.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, three points could not be checked: whether the hio.c and pruneheap.c logic runs only for direct TOAST tables, which lock the in-place upgrade takes, and whether prune-to-LP_UNUSED is safe next to indexed plain chunks.

## The proposal, restated

**Problem.** Every TOAST chunk needs a B-tree index insert and a lookup on read, which costs WAL, index bloat, two-pass vacuum and read latency. The author also cites the 32-bit TOAST OID limit, with one real table running out of OIDs at 1 billion rows. The target is large tables with many toasted columns and heavy update churn.

**Proposed design.** A new external pointer, varatt_direct (VARTAG_DIRECT), stores the TID of a root chunk instead of a value ID. Values up to one chunk are read directly, values up to 100 chunks keep a tid[] array in the last chunk, and larger values use a tree with fanout 50 and byte offsets for slice reads. Direct chunks have chunk_id NULL and stay out of a partial index, so dead chunks are pruned straight to LP_UNUSED.

**User-visible surface.**
- New table storage parameter toast_flavour ('plain' | 'direct'), settable with ALTER TABLE, which upgrades the TOAST table in place.
- New GUC toast_default_flavour, choosing the 3-column or 5-column format for new TOAST tables.
- New GUC and reloption direct_toast_self_prune (default on).
- New superuser function pg_ensure_direct_toast(regclass).
- Direct TOAST tables get columns chunk_tids and chunk_tid_offsets, a partial unique index, and refuse VACUUM FULL, CLUSTER and REPACK run on the TOAST table directly.

**What the author claims.**
- No migration: plain and direct pointers coexist in one table and the switch is metadata-only.
- A 25M-row table loaded about 2x faster, reached 1385 TPS against 635 TPS, and grew to 380GB against 500GB after 100 hours.
- pgvector top-N queries on toasted vectors ran 5% to 30% faster.
- Writes of 1 MB to 1 GB values ran 7% to 15% faster; reads are equal when the index is cached and faster on cold caches.
- TOAST WAL volume drops by about 40-50%, and vacuum skips index cleanup for direct chunks.

**Thread state.** Michael Paquier and Matthias van de Meent object to losing standalone VACUUM FULL/REPACK on TOAST tables, to the loss of chunk identity checks, and to skipping index cleanup; Paquier says he cannot support it. The author answers each point and posts v15, but no reviewer accepts the design.

## 2. Prior attempts

### Support for 8-byte TOAST values (aka the TOAST infinite loop problem) (related)

- **Link:** [https://commitfest.postgresql.org/patch/5830/](https://commitfest.postgresql.org/patch/5830/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aFOnKHG7Wn-Srnpv@paquier.xyz](https://www.postgresql.org/message-id/flat/aFOnKHG7Wn-Srnpv@paquier.xyz)
- **Date:** decided 2026-02-10; started 2025-06-19
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The toast_external layer with function pointer redirection got no support. Tom Lane questioned its run-time cost and called it overdesigned, and the author later said the design had not received a single +1. Only the oid8 data type was committed. The author then dropped the rest from the v19 cycle, planning a v20 rework without the redirections.
- **Reason (quote):** "Anyway, as far as things stand today on this thread, I have not been able to get even a single +1 for this design." ([message](https://www.postgresql.org/message-id/aTn4aS32O4lmaPnj%40paquier.xyz))
- **What is different this time:** P7 widened the value ID to 8 bytes and added a callback layer per vartag. Direct TOAST drops the value ID and the index lookup, and addresses chunks by TID. Only oid8 from P7 was committed, and Direct TOAST builds on it.


## 3. Objections a reviewer is likely to raise

### 1. Stale direct pointers can read another value's chunk without error

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, WAL / replication / standby · `replication-ondisk-1`

A DDL command scans with SnapshotAny and detoasts a recently dead tuple, as described in message #28. Its direct chunks were already pruned to LP_UNUSED, and a new insert reused the line pointer. Today this case raises a missing-chunk error, but with direct TOAST the backend returns another row's bytes and nobody notices.

Plain TOAST checks chunk_id and chunk_seq on every fetch, and messages #2, #21 and #28 describe real bugs that this check catches. A direct pointer carries only a TID, and the chunk carries no link to its value. Patch 0009 adds prune-to-LP_UNUSED, so line pointers get reused quickly and a stale TID is more likely to land on a live chunk. Message #25 offers to look into chunk numbers and a back-pointer, but v15 does not include either.

Thread: [#2 (Michael Paquier, 2026-09-07)](https://www.postgresql.org/message-id/ap4r6H0A-5SD8pGj%40paquier.xyz), [#21 (Matthias van de Meent, 2026-09-29)](https://www.postgresql.org/message-id/CAEze2WjaraCS9Ux9Q8aX38WaqHjvF5HnBM%2BoAPLiHUaZ7wjVpQ%40mail.gmail.com), [#22 (Michael Paquier, 2026-09-30)](https://www.postgresql.org/message-id/arx6V8dC3EhZrDBq%40paquier.xyz), [#25 (Hannu Krosing, 2026-09-30)](https://www.postgresql.org/message-id/CAMT0RQToBWs4h2S2FzdfES%2BGOYgHRWbeG%3DAxtnbRd6jDYJHHPQ%40mail.gmail.com), [#28 (Matthias van de Meent, 2026-10-05)](https://www.postgresql.org/message-id/CAEze2WjEdPEaReJQNPu6%2BNzbFBW9LBazJ6Ls%3DufeHk2v%2BsNqyw%40mail.gmail.com)

**Question:** What data in the direct pointer or the chunk lets a fetch detect a reused or wrong TID and raise an error instead of returning foreign data?

Counter-check: held. Message #28 describes the SnapshotAny detoast case as an active problem, and prune-to-LP_UNUSED makes line pointer reuse faster. A reused single chunk of the same size passes the size check from va_extinfo, and message #25 offers identity data only as future work.

### 2. Physical TIDs remove TOAST-only rewrite as a way to remove bloat

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 6 · on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-2`

A table with compact HOT-updated rows and churning jsonb columns builds up a bloated direct TOAST table. The DBA runs VACUUM FULL or REPACK on the TOAST table and gets an error. The only remaining option is to rewrite the main table and rebuild all its GIN, trigram or HNSW indexes.

The chunk location is stored in the parent tuple, so any move of a chunk requires rewriting the parent. Messages #21, #22 and #28 say that rewriting only the TOAST table is a feature users rely on. Message #28 asks how to shrink a TOAST table whose surviving chunks sit on its last pages, and message #29 answers with a per-row update script and a planned REPACK TOAST. That plan would still have to rewrite parent tuples.

Thread: [#2 (Michael Paquier, 2026-09-07)](https://www.postgresql.org/message-id/ap4r6H0A-5SD8pGj%40paquier.xyz), [#21 (Matthias van de Meent, 2026-09-29)](https://www.postgresql.org/message-id/CAEze2WjaraCS9Ux9Q8aX38WaqHjvF5HnBM%2BoAPLiHUaZ7wjVpQ%40mail.gmail.com), [#22 (Michael Paquier, 2026-09-30)](https://www.postgresql.org/message-id/arx6V8dC3EhZrDBq%40paquier.xyz), [#24 (Hannu Krosing, 2026-09-30)](https://www.postgresql.org/message-id/CAMT0RQQeDnxX%2Bzk94rcNdew7%2B_oM37Wsegyh4mTHqTJ%3D1YGVYQ%40mail.gmail.com), [#28 (Matthias van de Meent, 2026-10-05)](https://www.postgresql.org/message-id/CAEze2WjEdPEaReJQNPu6%2BNzbFBW9LBazJ6Ls%3DufeHk2v%2BsNqyw%40mail.gmail.com), [#29 (Hannu Krosing, 2026-10-05)](https://www.postgresql.org/message-id/CAMT0RQQhiFfhZGG%3DUdL-MS4QJWK0Ub-dCP7y8DYe_y6ENTkT6w%40mail.gmail.com)

**Question:** How does a planned REPACK TOAST move direct chunks without rewriting parent tuples, and what lock does it hold on the main table while it does so?

Counter-check: held. The limit follows from storing a TID in the parent, and the planned REPACK TOAST does not remove it. Since v12 it applies only to tables that opt in, and message #22 names opt-in as a possible path, so it is a tradeoff to justify.

### 3. A bespoke chunk tree duplicates B-tree work outside the index AM

fixable · level 2, rubric item 11 · design, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-3`

Committers improve B-tree storage, deduplication or vacuum, and TOAST does not benefit from it. The tid[] tree with fanout 50, int8 offsets, recursive delete, amcheck checks and reorderbuffer rebuild code then needs its own fixes for corruption, recovery and decoding bugs, and few people know that code.

Message #1 says the B-tree inside the TOAST table added more code than it removed. Message #28 describes direct TOAST as a specialized inline index and suggests the same effort could go into making the index path faster for TOAST. The read gain is not settled: message #8 says plain matched direct on cached 1 GB reads. Message #21 estimates about 5x fewer page reads, not 10x. No committer in the thread has offered to own the new structure.

Thread: [#1 (Hannu Krosing, 2026-09-05)](https://www.postgresql.org/message-id/CAMT0RQT5HHYLGWOWPh%3DsceKS7%2BjEE5jTULRTWC9aY9vYjF8OvA%40mail.gmail.com), [#8 (Hannu Krosing, 2026-09-07)](https://www.postgresql.org/message-id/CAMT0RQR2Fh8DQsOL%3DMv90CBv856pDry5TQTsT3KfEZ%3DY9%3DbJPA%40mail.gmail.com), [#21 (Matthias van de Meent, 2026-09-29)](https://www.postgresql.org/message-id/CAEze2WjaraCS9Ux9Q8aX38WaqHjvF5HnBM%2BoAPLiHUaZ7wjVpQ%40mail.gmail.com), [#22 (Michael Paquier, 2026-09-30)](https://www.postgresql.org/message-id/arx6V8dC3EhZrDBq%40paquier.xyz), [#28 (Matthias van de Meent, 2026-10-05)](https://www.postgresql.org/message-id/CAEze2WjEdPEaReJQNPu6%2BNzbFBW9LBazJ6Ls%3DufeHk2v%2BsNqyw%40mail.gmail.com)

**Question:** Which of the measured gains cannot be reached by a narrower B-tree or TOAST index optimization that keeps chunk identity and standalone REPACK?

Counter-check: held. Message #1 says the tree added more code than it removed, and no committer has offered to own it. A denser B-tree cannot remove index inserts or their WAL, so the author's answer should separate those gains from the read gains, which message #8 weakens.

### 4. Two pointer formats in one table add a branch to every TOAST consumer

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API, WAL / replication / standby · `maintenance-fit-2`

A future patch to heap rewrite, logical decoding or an extension handles VARTAG_ONDISK pointers and does not handle VARTAG_DIRECT. A direct pointer then reaches code that reads va_tid bytes as a value ID, and nothing fails until a user hits that path on real data.

Plain and direct pointers coexist in one table with no time limit, so every caller of VARATT_IS_EXTERNAL_ONDISK must handle both formats. Message #1 says the checks were placed next to the existing ones. v15 (message #31) fixed exactly this kind of bug: toast_preserve_valueid read va_tid as an OID after a flavour change. The proposal does not say what would make a missed branch fail loudly in cassert builds or in tests.

Thread: [#1 (Hannu Krosing, 2026-09-05)](https://www.postgresql.org/message-id/CAMT0RQT5HHYLGWOWPh%3DsceKS7%2BjEE5jTULRTWC9aY9vYjF8OvA%40mail.gmail.com), [#2 (Michael Paquier, 2026-09-07)](https://www.postgresql.org/message-id/ap4r6H0A-5SD8pGj%40paquier.xyz), [#31 (Hannu Krosing, 2026-10-18)](https://www.postgresql.org/message-id/CAMT0RQSSSq8-5DL6w3f%2Bdh1R9bAyGmega41OcDoQt_rKbMMFGQ%40mail.gmail.com)

**Question:** What mechanism makes code that handles only the plain on-disk pointer fail visibly when it receives a direct pointer?

Counter-check: held. Message #31 reports a fix for this kind of bug, where toast_preserve_valueid read va_tid bytes as an OID. The proposal names no assertion or test that catches the next missed branch.

### 5. Remaining demand rests on author-run benchmarks after OID8 TOAST was committed

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer pushes Direct TOAST mainly for the out-of-OIDs case, which OID8 TOAST already handles. Users of tables with heavy TOAST update churn turn it on for speed. The speedup turns out smaller than the thread claimed, and the project keeps a second TOAST format that few people use.

The only field case, a table that ran out of TOAST OIDs at 1 billion rows (message #3), is now covered by OID8 TOAST, which v6 rebased onto (message #11). What remains is performance, measured by the author alone. Message #2 notes the main benchmark has no configuration or scripts, and message #8 retracts the 2x cold-read result. No other user reports TOAST index overhead as a production problem, and message #4 is interest from one prior implementer.

Thread: [#2 (Michael Paquier, 2026-09-07)](https://www.postgresql.org/message-id/ap4r6H0A-5SD8pGj%40paquier.xyz), [#3 (Hannu Krosing, 2026-09-07)](https://www.postgresql.org/message-id/CAMT0RQQ9E74sOfs_7VdvFEdjSFFW6mOztt2PpmPziNWC6_z6%2Bw%40mail.gmail.com), [#4 (Nikita Malakhov, 2026-09-07)](https://www.postgresql.org/message-id/CAN-LCVM-rh9j_YbG93S7RUBbeu6-1TEWe4FJQrPqUvmR2cM2JQ%40mail.gmail.com), [#8 (Hannu Krosing, 2026-09-07)](https://www.postgresql.org/message-id/CAMT0RQR2Fh8DQsOL%3DMv90CBv856pDry5TQTsT3KfEZ%3DY9%3DbJPA%40mail.gmail.com), [#11 (Hannu Krosing, 2026-09-21)](https://www.postgresql.org/message-id/CAMT0RQQvk50Gz-5o%3DvU7oqmK1gXzzVKiMJiwrkUczETTPibAyA%40mail.gmail.com)

**Question:** Which production workloads already on OID8 TOAST show TOAST index cost as a bottleneck, measured with published, reproducible scripts?

Counter-check: held. Messages #11 and #26 confirm that oid8 TOAST handles OID exhaustion, so the case rests on performance. The reported gains are large (1385 against 635 TPS), but message #2 notes they come without configuration or scripts, and no second user reports the problem.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | New on-disk pointer tag VARTAG_DIRECT and a 5-column TOAST table format with a partial index. |
| WAL / replication / standby | **yes** | Patch 0004 changes decode.c, reorderbuffer.c, proto.c and pgoutput.c to reassemble direct chunks by TID. |
| wire protocol | no |  |
| catalog changes | **yes** | TOAST tables gain two columns, the primary key becomes a partial unique index in place, and pg_proc gains a function. |
| ABI / extension API | **yes** | varatt.h adds a vartag that extensions and output plugins testing VARATT_IS_EXTERNAL_ONDISK do not know. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Adds toast_default_flavour and direct_toast_self_prune, plus the toast_flavour reloption. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump --binary-upgrade must recreate a 5-column TOAST table even when the reloption or GUC says 'plain'. |
| psql UX conventions | no |  |
| security / privileges | **yes** | Adds superuser function pg_ensure_direct_toast; a reused TID could return bytes of another column or row. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second TOAST format with its own chunk tree, delete, decoding and amcheck code stays as long as data uses it. |
| performance on the hot path | **yes** | Changes detoast.c for all TOAST reads, and message #31 lists changes in hio.c and pruneheap.c. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | oid8 TOAST already covers OID exhaustion, and message #28 suggests improving the TOAST index path instead. |

## 5. Questions the author must answer before anyone reviews code

1. What data in the direct pointer or the chunk lets a fetch detect a reused or wrong TID and raise an error, including for single-chunk values?
2. How does a user remove bloat from a direct TOAST table with core commands, without rewriting the main table and rebuilding its indexes?
3. With oid8 TOAST as the baseline, which gains remain under published configuration and scripts, and which of them could a TOAST-specific B-tree improvement not reach?
4. What does pg_dump --binary-upgrade record so that a 5-column TOAST table with a partial index survives pg_upgrade when toast_flavour is unset or 'plain'?
5. What lock does the in-place change of pg_index and pg_constraint take, and what happens to concurrent writers on the table during it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier, author of the 8-byte TOAST work (P7), takes part but says he cannot support the design. An owner would need to know heap pruning, TOAST and logical decoding.
- **Liability:** A permanent on-disk vartag and TOAST table format, a chunk tree with its own delete, decoding and amcheck code, and pruning changes in shared heap paths. Bugs here return wrong data or lose data.
- **Author follow-up:** Strong sign: the author answered each review point and posted versions up to v15 in about two months.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The target is large tables with many toasted columns and heavy update churn. The series adds an on-disk format across TOAST, heap pruning, logical decoding and amcheck, and reviewers reject the loss of chunk identity and of standalone TOAST repack.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) TOAST table format is not captured by pg_dump --binary-upgrade. Not listed: not among the strongest objections.
- `standards-compat-2` (fixable, level 2) toast_default_flavour changes catalog shape without changing write behaviour. Not listed: not among the strongest objections.
- `standards-compat-3` (fixable, level 2) One-way format change disables existing TOAST maintenance commands. Not listed: not among the strongest objections.
- `replication-ondisk-3` (fixable, level 2) Third-party decoding plugins and extensions do not know VARTAG_DIRECT. Not listed: not among the strongest objections.
- `maintenance-fit-4` (fixable, level 2) Self-pruning logic enters heap prune and insert paths for all tables. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` Committed oid8 TOAST already covers the OID exhaustion motivation. Refuted (not specific): Duplicate of demand-1, which makes the same point about oid8 TOAST and adds the missing benchmark details. Message #26 already drops OID exhaustion as the main argument and rests the case on index cost.
- `security-1` Reused TID can return another row's data instead of an error. Refuted (not specific): Duplicate of replication-ondisk-1. The privilege and RLS exposure is a consequence of the same missing identity check, and the same fix answers both.
- `maintenance-fit-1` TID-pinned TOAST values make the TOAST table impossible to rewrite on its own. Refuted (not specific): Duplicate of replication-ondisk-2, which also notes that the planned REPACK TOAST would still rewrite parent tuples.
- `already-exists-1` Second chunk-lookup mechanism next to the TOAST B-tree index. Refuted (not specific): Duplicate of maintenance-fit-3, which raises the same inline-index point and asks the same question about a narrower B-tree optimization.
- `already-exists-2` OID exhaustion motivation is already met by committed OID8 TOAST. Refuted (not specific): Duplicate of demand-1, which covers the oid8 commit and the unreproducible benchmarks in more detail.
- `demand-2` Everyone pays for a second TOAST format; only opt-in tables gain. Refuted (not specific): Duplicate of maintenance-fit-2 for the missed-branch scenario and of maintenance-fit-3 for the B-tree alternative. It adds no separate failure case.

### Review angles

- prior-attempts: 1 objection(s). P1 (64-bit IDs, closed when the patch went stale) and P11 (pluggable toaster, no recorded reason) give no reason that applies here. The thread mentions an earlier Direct TOAST discussion (message #1) and overflow pages (messages #22, #29), but reports no rejection reason for either.
- standards-compat: 3 objection(s). Objections were found; this note does not apply.
- replication-ondisk: 3 objection(s). Not applicable.
- security: 1 objection(s). Other privilege points look settled: pg_ensure_direct_toast is superuser-only since v10, and toast_flavour needs table ownership like other reloptions.
- maintenance-fit: 4 objection(s). Objections found; see list.
- already-exists: 2 objection(s). Two objections apply from this angle. No extension or external tool provides TID-addressed TOAST, and the pluggable toaster route [P11] was rejected, so an extension path is not proposed here.
- demand: 2 objection(s). Two demand objections raised; design and safety concerns (identity checks, standalone REPACK of TOAST, index cleanup) belong to other angles.

### Precedents retrieved

- P1: [64-bit TOAST value ID](https://commitfest.postgresql.org/patch/4296/) (cf, Returned with feedback, 2023-09-01) - used in the review; reason summarised from its thread
- P2: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P3: [support tid range scan in parallel](https://commitfest.postgresql.org/patch/4982/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P4: [Tid range scan](https://commitfest.postgresql.org/patch/1755/) (cf, Returned with feedback, 2019-12-01) - judged not relevant; reason summarised from its thread
- P5: [Cleanup gcc trick with varattrib_1b_e in VARATT_EXTERNAL_GET_POINTER()](https://commitfest.postgresql.org/patch/5807/) (cf, Withdrawn, 2025-07-14) - judged not relevant; reason summarised from its thread
- P6: [Tidy fill hstv array (src/backend/access/heap/pruneheap.c)](https://commitfest.postgresql.org/patch/4734/) (cf, Returned with feedback, 2024-03-04) - judged not relevant; reason summarised from its thread
- P7: [Support for 8-byte TOAST values (aka the TOAST infinite loop problem)](https://commitfest.postgresql.org/patch/5830/) (cf, Returned with feedback, 2026-02-10) - used in the review; reason summarised from its thread
- P8: [Revert "Tidy up locale thread safety in ECPG library."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3c8e463b0d885e0d976f6a13a1fb78187b25c86f) (revert, Reverted, 2025-03-28) - judged not relevant
- P9: [Avoid possible null pointer dereference](https://commitfest.postgresql.org/patch/5506/) (cf, Withdrawn, 2025-02-06) - judged not relevant; reason summarised from its thread
- P10: [Refuse setting toast.* reloptions when TOAST relation does not exist](https://commitfest.postgresql.org/patch/1486/) (cf, Returned with feedback, 2018-04-10) - judged not relevant; reason summarised from its thread
- P11: [Pluggable toaster](https://commitfest.postgresql.org/patch/3490/) (cf, Rejected, 2023-04-03) - used in the review
- P12: [pg_stat_toast](https://commitfest.postgresql.org/patch/3457/) (cf, Withdrawn, 2022-07-01) - judged not relevant
- P13: [Detect escape of ErrorContextCallback stack pointers (and from PG_TRY() )](https://commitfest.postgresql.org/patch/2722/) (cf, Withdrawn, 2020-09-10) - judged not relevant
- P14: [Eliminating CREATE INDEX comparator TID tie-breaker overhead](https://commitfest.postgresql.org/patch/312/) (cf, Rejected, 2015-07-30) - judged not relevant
- P15: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P16: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P17: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P18: [64-bit XIDs](https://commitfest.postgresql.org/patch/3594/) (cf, Returned with feedback, 2023-12-15) - judged not relevant
- P19: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P20: [Revert "Fix typo in documentation of toast storage"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=664f01b61326eebc0d7b4ce9c89b99d3f18c3b76) (revert, Reverted, 2018-11-26) - judged not relevant
- P21: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P22: [Removing unused trailing linepointers from heap pages](https://commitfest.postgresql.org/patch/3035/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P23: [64-bit XIDs](https://commitfest.postgresql.org/patch/4703/) (cf, Returned with feedback, 2024-09-16) - judged not relevant
- P24: [Skip all-visible pages during second HeapScan of CIC](https://commitfest.postgresql.org/patch/1057/) (cf, Returned with feedback, 2017-03-24) - judged not relevant

### Warnings

- attachment limit (8) reached; v15-0009-Prune-dead-unindexed-TOAST-tuples-directly-to-LP.patch not read
- attachment limit (8) reached; v15-0010-Add-direct_toast_self_prune-vectored-detoasting-.patch not read
- attachment limit (8) reached; v2-0001-Refactor-detoasting-pipeline-to-unify-full-and-sl.patch not read
- attachment limit (8) reached; v2-0005-Add-amcheck-verification-for-Direct-TOAST-tuples.patch not read
- attachment limit (8) reached; v2-0004-Support-Direct-TOAST-in-logical-decoding-replicat.patch not read
- attachment limit (8) reached; v2-0003-Implement-Direct-TOAST-core-storage-reading-and-w.patch not read
- attachment limit (8) reached; v2-0006-Add-documentation-for-Direct-TOAST.patch not read
- attachment limit (8) reached; v2-0002-Add-Direct-TOAST-catalog-GUC-and-reloptions-infra.patch not read
- attachment limit (8) reached; v2-0007-Add-pg_ensure_direct_toast-for-in-place-legacy-TO.patch not read
- attachment limit (8) reached; v2-0008-Add-backend-TOAST-architecture-documentation-and-.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 248 s; cost reported by the CLI: $5.98
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 135 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:34:30.817Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
