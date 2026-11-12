# Design pre-review: Restructured Shared Buffer Hash Table

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7326](https://commitfest.postgresql.org/patch/7326/); [thread](https://www.postgresql.org/message-id/flat/CAAStW3JX_7LcCD1U%2B89vyMmLreDs0XsBU-JDdFf5PD_Woy1pag%40mail.gmail.com), 30 of 30 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The decisive objection is replication-ondisk-1. The flat table needs slot x unlinked before it is reused, message #12 rejects the spinlock-held delete that guarantees this, and no posted version carries an agreed replacement. maintenance-fit-1 adds that no check catches a later path that breaks the rule. demand-1 and prior-attempts-1 are open but do not stop the idea, since message #2 supports a separate table and the micro-benchmarks and memory saving are real.

Not checked: Only 3 of 30 messages and no code were read, so the v6 two-step delete could not be checked against the slot-reuse race. Whether the partition lock already gives the needed exclusion on some paths could not be judged.

## Prior attempts

- [drop/truncate table sucks for large values of shared buffers](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com) (related; Returned with feedback, 2015-07-02). "Committing a band-aid will just make it harder to pursue real fixes." ([message](https://www.postgresql.org/message-id/14561.1435845813%40sss.pgh.pa.us)) Different now: P3 added a per-block lookup path for DROP/TRUNCATE, and this proposal replaces the table for every lookup. P3's reasons concern that path and do not transfer; the shared point is the stated preference for a radix-tree structure.

## Objections a reviewer is likely to raise

### 1. Slot-reuse invariant has no agreed safe design, and a miss means duplicate buffers

InvalidateBuffer() releases buffer x, and a concurrent backend reuses slot x before the old entry is unlinked. The bucket chain breaks, so a later BufTableLookup() misses a block that is still cached. A second buffer is then loaded for the same page, one copy's changes are lost at write-out, and the loss shows up only later as heap or index corruption, on a primary or during standby replay.

Because entries[x] must describe buffer x, a stale entry and a new entry cannot exist at the same time (message #16). The v1 fix holds the buffer header spinlock across BufTableDelete(). Andres rejects that in message #12, since an error there cannot be recovered and it risks deadlock with the partition lock. The v6 two-step delete (message #18) comes with no argument for why the invariant holds, and the CFBot failures in 027_stream_regress (message #24) show that this test does catch races in this table.

Precedent: [BufferAlloc: don't take two simultaneous locks](https://www.postgresql.org/message-id/flat/1edbb61981fe1d99c3f20e3d56d6c88999f4227c.camel@postgrespro.ru) (Returned with feedback, 2022-10-12)

**Question:** Without the spinlock held across BufTableDelete(), what ordering of buffer header lock and partition lock stops slot x from being reused before its old entry is unlinked, including when an error is raised partway?

<sub>fixable, level 2, design; messages [#12](https://www.postgresql.org/message-id/bk5cbayf3q4yll4fqs2wbmonzhykg43k2k5v37vokkko53ebsd%40spcp4mxlb7hb), [#16](https://www.postgresql.org/message-id/CAAStW3KnhSSMywM0kTLjOQOxhm1%2B0jFEh%2BSbZMuMytRYGzLcmg%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/CAE8JnxOPr1JZ9xhhbgyAnTt3qFmtNO-OAwoxN_sq_ug1WWma9A%40mail.gmail.com), [#24](https://www.postgresql.org/message-id/CAE8JnxNGtaKUs9ZqkcPhWcEJ1RFfGZHJOAqXiqTGQzCe5gv97g%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/CAE8JnxOcd%2BJ0A8iM%3DBwwhovXoih5AOtvUcxPRbqgtn9CNAjPXQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Slot-reuse invariant adds a silent rule for every buffer-reuse path

A later patch adds a new path that evicts or reuses a buffer, for example shrinking shared buffers online, and releases the header lock before the mapping delete. A concurrent backend reuses slot x and cuts the old chain, so lookups miss cached pages and the same page can be read into a second buffer. Nothing asserts, and the damage shows up later as corruption.

Dynahash tolerates a stale entry and a new entry at once because it allocates from a freelist (message #16), but the flat table does not. v1 closes the race by holding the header spinlock across BufTableDelete(), which message #12 rejects as unsafe, since spinlocks have no error recovery and the change risks deadlock. The lock-free variant failed 027_stream_regress on CFBot (message #24) and was withdrawn (message #25). No posted version states the replacement protocol or how a violation would be detected.

**Question:** What protocol replaces the spinlock-held delete, and what check makes a future reuse path fail loudly if it breaks the entries[x] invariant?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#12](https://www.postgresql.org/message-id/bk5cbayf3q4yll4fqs2wbmonzhykg43k2k5v37vokkko53ebsd%40spcp4mxlb7hb), [#16](https://www.postgresql.org/message-id/CAAStW3KnhSSMywM0kTLjOQOxhm1%2B0jFEh%2BSbZMuMytRYGzLcmg%40mail.gmail.com), [#24](https://www.postgresql.org/message-id/CAE8JnxNGtaKUs9ZqkcPhWcEJ1RFfGZHJOAqXiqTGQzCe5gv97g%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/CAE8JnxOcd%2BJ0A8iM%3DBwwhovXoih5AOtvUcxPRbqgtn9CNAjPXQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. No workload-level gain shown to justify a dedicated hash table

A committer pushes a custom buffer mapping table on micro-benchmark numbers. Users running pgbench-like or production workloads see no TPS change, but every later bufmgr patch has to respect the new slot-reuse invariant. The project finds out when a buffer-mapping redesign has to replace or work around it.

The only end-to-end data shows no significant TPS change: the author's 30-run rerun has confidence intervals spanning zero in most cells (message #10), and Andrey saw a possible simple-update regression (message #9). The brief targets large buffer pools, but the speedup shrinks to about 1.02x for insert and delete at 16GB (message #1, message #27). Heikki asked for real-world impact (message #2), and no field report or profile with numbers has answered it. Andres questions spending review bandwidth here instead of on the architectural problems (message #12).

Precedent: [drop/truncate table sucks for large values of shared buffers](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com) (Returned with feedback, 2015-07-02)

**Question:** Which real workload, at which shared_buffers size, spends measurable CPU in BufTableLookup today and shows a repeatable end-to-end gain with this patch?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAAStW3JX_7LcCD1U%2B89vyMmLreDs0XsBU-JDdFf5PD_Woy1pag%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/dbbd1998-19ff-4ac2-b4b1-a39f4ec1b0f5%40iki.fi), [#9](https://www.postgresql.org/message-id/FE84530D-D06E-4317-A999-CE6F433E6CE3%40yandex-team.ru), [#10](https://www.postgresql.org/message-id/CAAStW3LMwY%3DSW7GLmn81mzQGH%3DbH%2BdwZZCrsfNbM9b7eiptLjQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/bk5cbayf3q4yll4fqs2wbmonzhykg43k2k5v37vokkko53ebsd%40spcp4mxlb7hb), [#27](https://www.postgresql.org/message-id/CAAStW3KitUuDbLB17N53aGj-X89unTRpV1BYYqjSc-UV%2BXz-5w%40mail.gmail.com); `demand-1`</sub>

### 4. Tuning the hash table may delay the buffer mapping redesign

The flat table is committed, and the linear-hashing resize from message #7 is then built on it for dynamic shared buffers. Later, someone starts the relation-plus-block radix tree from message #12 and finds two hash-specific layers to replace. The constraint that entries[x] describes buffer x now ties bufmgr.c to the old layout.

[P3] was returned partly because a band-aid was seen to make a structure that finds all buffers of a relation harder to pursue. Andres repeats that point here in message #12: per-relation iteration, ordered lookup and 20-byte keys, plus doubt about spending effort on the current design. Message #25 answers with cost estimates for a radix tree, but the thread did not settle the point. The proposal does not say whether the flat table is a step toward the redesign or a separate path.

Precedent: [drop/truncate table sucks for large values of shared buffers](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com) (Returned with feedback, 2015-07-02)

**Question:** Does the flat table, with its planned resize, make the per-relation, ordered buffer mapping in message #12 any easier or harder to build later?

<sub>fixable, level 1b, prior rejection; messages [#7](https://www.postgresql.org/message-id/CAAStW3Kq5_55X_G74P%3DaovoDBuUEeE_pionGVQ6F_dS7BThLqg%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/bk5cbayf3q4yll4fqs2wbmonzhykg43k2k5v37vokkko53ebsd%40spcp4mxlb7hb), [#25](https://www.postgresql.org/message-id/CAE8JnxOcd%2BJ0A8iM%3DBwwhovXoih5AOtvUcxPRbqgtn9CNAjPXQ%40mail.gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. Without the spinlock held across BufTableDelete(), what ordering of buffer header lock and partition lock stops slot x from being reused before its old entry is unlinked?
2. Which patch version is under review, the v1 from message #1 or the v6 with two-step delete, and does the author agree with the v6 approach?
3. What assertion or check makes a future buffer-reuse path, such as shrinking shared buffers online, fail loudly if it breaks the entries[x] invariant?
4. Which real workload, at which shared_buffers size, shows a repeatable end-to-end gain with this patch, and is the simple-update regression from message #9 reproducible?
5. Does the flat table, with its planned linear-hashing resize, make the per-relation, ordered buffer mapping from message #12 easier or harder to build later?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-1: Slot-reuse invariant has no agreed safe design, and a miss means duplicate buffers
- **WAL / replication / standby:** see objection replication-ondisk-1: Slot-reuse invariant has no agreed safe design, and a miss means duplicate buffers
- **maintenance burden ("who maintains this in 10 years"):** buf_table.c gets its own hash table, and bufmgr.c gets a new ordering rule for every path that reuses a buffer.
- **performance on the hot path:** The patch rewrites BufTableLookup, BufTableInsert and BufTableDelete and changes buffer header lock hold time in InvalidateBuffer().

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas and Andres Freund replied in the thread; the owner must know bufmgr.c locking, buffer invalidation and eviction.
- Priority: fix design first; audience unclear; review cost high. Every workload uses the lookup path, but pgbench shows no significant TPS change, so who gains measurably is not shown. The patch is small, but the review must prove a concurrency invariant in bufmgr.c, and the delete protocol is not yet agreed.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
