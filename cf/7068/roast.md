# Design pre-review: Lost KEY SHARE lock information during toasting allows foreign-key violation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7068](https://commitfest.postgresql.org/patch/7068/); [thread](https://www.postgresql.org/message-id/flat/CA%2BCOZaC-2-GQabq%2Bem11PVdAX6H%2BvFYvU6mWC%3DQyNHb1RBuczg%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real and the fix is wanted: message #1 shows an orphaned child row from ordinary DML. Two fixable design points survive at level 2. replication-ondisk-1 shows v1 writes a lock-only xmax naming the updater and a duplicate multixact member, and maintenance-fit-1 shows the recheck condition already missed one unlock path. Rui's follow-up patches address both, but no merged version exists and nobody has answered message #4.

Not checked: The code was not read, so the handling of locker_remains and checked_lockers, the page-extension path and the exact headers written by the merged version could not be checked. The end of message #4 is cut off.

## Prior attempts

None found before 2026-07-28.

## Objections a reviewer is likely to raise

### 1. v1 persists wrong xmax and multixact state on disk and on standbys

A minor release ships v1. A plain update of a row with an external value stamps the new tuple with a lock-only key-share xmax naming the updater itself. Updates that meet a real key-share locker create multixacts holding the updater twice. Both forms are WAL-logged, replayed on standbys and kept through pg_upgrade, and nobody notices until the headers confuse a later lock check or pg_multixact members grow faster.

Message #4 shows the new tuple's xmax at the updater's own xid on v1, where master writes 0. It also shows an extra fornokeyupd member in both multixacts, and a caller's tuple whose header no longer matches the stored one. These headers live in heap pages and pg_multixact and pass to standbys through the update WAL record, so a stable-branch fix cannot take them back once released. Nobody has answered message #4, and message #3 says the locker_remains logic is unsettled.

**Question:** Which xmax and infomask invariants does the merged version keep on the old and new tuple, and does replay produce the same headers as the primary?

<sub>fixable, level 2, WAL / replication / standby; messages [#3](https://www.postgresql.org/message-id/aml62IuY_wUS5zvi%40paquier.xyz), [#4](https://www.postgresql.org/message-id/CAHWVJhGtxkEhiKU7ntFGQ2Lw9kUTebJD3N-xR8Q9OXem37A%2BUg%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Recheck is tied to a proxy condition, so later unlock paths can skip it

A later patch adds another path in heap_update() that drops the buffer lock, or changes when the toaster returns its input tuple. The xmax recheck does not run on that path, and a concurrent KEY SHARE lock is lost again. Nothing fails until someone finds an orphaned child row in production.

Message #4 shows the v1 recheck already misses one unlock path. v1 gates on `heaptup != newtup || newbuf != buffer`, but the unlock happens under `need_toast || newtupsize > pagefree`. Rui's 0001 realigns the two conditions, yet the design still needs every future edit of this block to keep the conditions in step. Only an injection-point test for one path would fail loudly if they drift.

**Question:** Can the recheck be keyed on a single record that the buffer lock was released, so any new unlock path in heap_update() is covered without a separate change?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/aml62IuY_wUS5zvi%40paquier.xyz), [#4](https://www.postgresql.org/message-id/CAHWVJhGtxkEhiKU7ntFGQ2Lw9kUTebJD3N-xR8Q9OXem37A%2BUg%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which xmax and infomask values does the merged version write on the old and new tuple, and does replay on a standby produce the same headers?
2. Can the recheck be keyed on one record that heap_update() released the buffer lock, so a new unlock path is covered without a separate change?
3. Which back branches have the bug, and does the same patch apply to each of them?
4. Does page extension without TOAST work open the same window, and does a test cover that path?
5. How should locker_remains and checked_lockers be set when the xmax changed while the buffer lock was released?

## Touch points

- **on-disk format / pg_upgrade:** The patch changes which xmax and infomask values heap_update() stamps on heap tuples, and which multixact members it creates.
- **WAL / replication / standby:** The tuple headers written by heap_update() reach standbys through the update WAL record; no new record type is added.
- **backpatchability:** The CommitFest target is stable, but the reporter tested master only and the affected back branches are not listed.
- **maintenance burden ("who maintains this in 10 years"):** The xmax recheck in heap_update() must stay in step with every path that releases the buffer lock.
- **performance on the hot path:** The patch reorders work inside heap_update(); the extra recheck runs only when the buffer lock was released.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier wrote v1 and asked for more review; a committer who knows heapam tuple locking and multixacts should also confirm the merged version.
- Priority: proceed; audience a sizeable minority; review cost high. Users of foreign keys whose referenced tables have TOASTed columns can get orphaned child rows from ordinary DML (message #1). The patch is small, but it changes tuple locking in heap_update() for stable branches, so review needs care and one merged, rebased version.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
