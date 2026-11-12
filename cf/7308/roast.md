# Design pre-review: Optimize truncation logic to reduce AccessExclusive lock impact

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7308](https://commitfest.postgresql.org/patch/7308/); [thread](https://www.postgresql.org/message-id/flat/CA%2BYyo5Trs9x03-sc4PpmEXPY9K_B_jbEbNd01RbF%2BvOxs3zVUA%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the lock problem is real and reported from production. Patch 2 makes the buffer drop on a primary depend on an exact fork size (prior-attempts-1, maintenance-fit-2), which is the doubt recorded for [P1]. Someone who knows the buffer manager has to judge that. Patch 1 still needs a workload that justifies it once patch 2 is applied (maintenance-fit-1).

Not checked: The material has no prose or file list for patch 2, so its API shape and release-build checks could not be judged. The safety argument is known only from the objections' account of message #5.

## Prior attempts

- [drop/truncate table sucks for large values of shared buffers](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com) (similar idea; Returned with feedback, 2015-07-02). "Committing a band-aid will just make it harder to pursue real fixes." ([message](https://www.postgresql.org/message-id/14561.1435845813%40sss.pgh.pa.us)) Different now: P1 aimed at DROP and TRUNCATE TABLE. Patch 2 covers only relation truncation and reuses the old_nblocks values that smgrtruncate() already has, with the existing FindAndDropRelationBuffers() path and threshold.

## Objections a reviewer is likely to raise

### 1. Wrong relation size from lseek would leave buffers behind

On a primary, vacuum truncates a table tail and smgrnblocks() returns a size that is too small, for example because of a kernel or file system bug. The targeted drop skips dirty buffers past that size. The checkpointer later tries to write a block of a truncated file and fails, and nobody sees it until then.

[P1] tried the same per-block lookup and, according to its thread summary, was turned down partly because it fails badly if the caller is wrong about the relation's last page. Message #5 argues only that the relation cannot grow between measuring and dropping. It does not cover a wrong size returned by lseek. The full scan does not depend on the reported size. d6ad34f34 limited the shortcut to recovery, where the size is cached, and this patch extends it to normal running.

Precedent: [drop/truncate table sucks for large values of shared buffers](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com) (Returned with feedback, 2015-07-02)

**Question:** Outside recovery, what happens if smgrnblocks() underreports the fork size, and does the existing truncation path already fail in that case without the full scan?

<sub>fixable, level 2, prior rejection; messages [#2](https://www.postgresql.org/message-id/CAApHDvrfngqCpQ09LKdr-BnJEVHW0%3DwAhxfRBnw%3DHJ2645cJPg%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAGRkXqRcNOrciQy%3D8sAYF%3D8Dsin2tYNpnUwe6V5JM%2BvXKeUHeA%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Targeted drop makes truncate callers responsible for exact fork sizes

A later patch or an extension calls smgrtruncate() with stale fork sizes, or while the relation can still grow. Buffers past the passed size stay in shared buffers, and the checkpointer later writes a dirty one past the new end of file. A release build gives no error at truncate time, so the problem shows up later as a checkpoint failure or a damaged relation.

Outside recovery, today's full scan does not depend on any size the caller passes. Message #5 moves correctness to the caller: the sizes must be exact and the relation must not grow, and only an Assert in RelationTruncate() enforces this. Outside recovery the size comes from lseek, and [P1] records doubt about trusting it as a reason the same lookup approach was not accepted. Reusing the existing FindAndDropRelationBuffers() path lowers the risk, but the proposal does not say what a release build checks.

Precedent: [drop/truncate table sucks for large values of shared buffers](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com) (Returned with feedback, 2015-07-02)

**Question:** In a release build, what detects a dirty buffer left past the passed size, and why is the lseek-accuracy doubt recorded for the earlier attempt not a concern here?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/CAApHDvrfngqCpQ09LKdr-BnJEVHW0%3DwAhxfRBnw%3DHJ2645cJPg%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAGRkXqRcNOrciQy%3D8sAYF%3D8Dsin2tYNpnUwe6V5JM%2BvXKeUHeA%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Patch 1 lets large tables keep much more empty tail space

A user with a 2 TiB table deletes about 40 GiB of rows at its end. Today vacuum truncates once 1000 free tail pages (8 MiB) exist. Under 3% plus 2 pages it needs about 60 GiB, so the space never goes back to the OS, and the user finds out when disk usage does not drop.

Patch 1 changes a constant that applies to every installation, so all users of large tables pay in disk space to cut lock time on hosts with very large shared_buffers. Message #1 shows no data on how often truncation runs or how long the lock is held. Patch 2 already removes most of the scan cost for small truncations, which weakens the case for patch 1. Users who want fewer truncations can already set the vacuum_truncate reloption per table.

**Question:** With patch 2 applied, which workload still needs the 3% plus 2 pages rule, and what lock times show it?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CA%2BYyo5Trs9x03-sc4PpmEXPY9K_B_jbEbNd01RbF%2BvOxs3zVUA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAApHDvrfngqCpQ09LKdr-BnJEVHW0%3DwAhxfRBnw%3DHJ2645cJPg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/0be607a5-20f6-4894-bb22-412e3c784bc2%40postgrespro.ru), [#5](https://www.postgresql.org/message-id/CAGRkXqRcNOrciQy%3D8sAYF%3D8Dsin2tYNpnUwe6V5JM%2BvXKeUHeA%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which patch is the CommitFest entry asking reviewers to look at: patch 2 alone, or both patches?
2. Outside recovery, what happens if smgrnblocks() underreports the fork size, and does the current truncation path already fail in that case without the full scan?
3. In a release build, what detects a dirty buffer left past the size passed to DropRelationBuffers()?
4. With patch 2 applied, which workload still needs the 3% plus 2 pages rule, and what lock times show it?
5. Does the lock problem remain for truncations above BUF_DROP_FULL_SCAN_THRESHOLD, and is that case in scope?

## Touch points

- **on-disk format / pg_upgrade:** Patch 2 changes which buffers are dropped at truncation, so a missed dirty buffer could be written past the new end of file.
- **ABI / extension API:** Patch 2 passes old fork sizes to DropRelationBuffers() and adds a lock assertion in RelationTruncate().
- **maintenance burden ("who maintains this in 10 years"):** Patch 2 makes callers of smgrtruncate() responsible for exact fork sizes; patch 1 changes the default truncation rule for all tables.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Patch 1 overlaps the existing vacuum_truncate reloption and VACUUM TRUNCATE option.

## Who would commit it, and when

- Owner: silent. Someone who knows the buffer manager and smgr truncation code would have to own patch 2. Patch 2 has no replies yet.
- Priority: proceed; audience a sizeable minority; review cost medium. Patch 2 reuses an existing lookup path and helps installations with large shared_buffers, with measurements shown. Review cost is medium because the patch 2 code was not read and it touches the buffer manager; patch 1 can wait for a stated workload.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
