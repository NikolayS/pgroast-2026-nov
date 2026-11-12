# Design pre-review: tablecmds: fix Deadlock caused by Attach Partition

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6442](https://commitfest.postgresql.org/patch/6442/); [thread](https://www.postgresql.org/message-id/flat/CFACA0EB-7E6F-4FAA-9ACE-1FC2226D7482%40gmail.com), 16 of 16 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is sound and 0001 adds only AccessShareLock on ancestors, which master already takes with a cold cache. The design of 0002 is open: standards-compat-1 and replication-ondisk-1 show it takes AccessExclusiveLock on ancestor tables and indexes for every multi-level index ATTACH, and no committer has accepted that cost. maintenance-fit-2 shows other bottom-up paths remain, and the author now holds the patch for a general fix.

Not checked: Without the code, this review could not check whether the callback releases ancestor locks correctly across lookup retries, or how it handles a target reparented before it is locked. It also could not check the order of permission checks and ancestor locks.

## Prior attempts

None found before 2026-01-30.

## Objections a reviewer is likely to raise

### 1. Index ATTACH now takes AccessExclusiveLock on the root table every time

A DBA attaches leaf indexes one by one on a busy multi-level tree, and most of these attaches validate nothing. With 0002, each attach takes AccessExclusiveLock on the root table and the ancestor indexes. Every query on the root then waits until the attach commits, and the DBA sees this only as an outage in production.

Message #14 measured this. Master locks no ancestors when the parent index stays invalid, and v5 takes AccessExclusiveLock on root, mid1 and their indexes. The same message gives the reason a weaker lock does not work, and asks a committer to accept the trade-off, which no committer has done. The entry is tagged Bugfix, so this lock change could also reach minor releases, and the patch prose touches no documentation of the new lock set.

**Question:** Can the deadlock be avoided without AccessExclusiveLock on the root when the attach will not validate any ancestor index, and if not, will the documentation state the new lock?

<sub>fixable, level 2, design; messages [#14](https://www.postgresql.org/message-id/CAOVWO5pmhdGfo7mvjheg1kryjMXNumDOiN9%3Dio2nthO%3DomRRNA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. New AccessExclusiveLock in index ATTACH also blocks hot standby readers

A DBA attaches a leaf index in a large partition tree where the parent index stays invalid. The primary WAL-logs the new AccessExclusiveLock on the root table and its index, and the standby replays it. Reporting queries on the root at the standby then wait or are cancelled after max_standby_streaming_delay, although master took no ancestor locks for this ATTACH.

Message #14 shows that v5-0002 takes AccessExclusiveLock on root, mid1, root_idx and mid1_idx even when nothing is validated, where master takes none. Postgres WAL-logs AccessExclusiveLock so hot standby can replay it, so this cost reaches every standby. The discussion in message #14 covers only blocking on the primary, and that question is still open. A committer would likely want the standby effect weighed in the same trade-off.

**Question:** How does the extra AccessExclusiveLock affect queries on hot standbys when no validation follows, and can the non-cascading case avoid it?

<sub>fixable, level 2, WAL / replication / standby; messages [#14](https://www.postgresql.org/message-id/CAOVWO5pmhdGfo7mvjheg1kryjMXNumDOiN9%3Dio2nthO%3DomRRNA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. Point fix leaves the bottom-up locking pattern open for future callers

A later patch adds a new caller of generate_partition_qual() or another child-then-parent walk, and does not call lock_partition_ancestors() first. No test or assertion fails. Users see occasional deadlocks against recursive ALTER TABLE in production, and a new thread starts to hunt for the cause.

After v5, the thread found the same inversion in pg_get_partition_constraintdef(), which psql \d+ uses (message #15), and in DROP NOT NULL (message #16). The author now says these should not be fixed one at a time and is holding the patch. Committing 0001 and 0002 as they stand adds a convention that every bottom-up path must follow, with nothing to enforce it, and a later general redesign may have to undo or rework them.

**Question:** Which rule, enforced in code or by tests, keeps the next caller that walks a partition up to its ancestors from bringing the deadlock back?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#12](https://www.postgresql.org/message-id/at3MCB1EIOHMjh6K%40alvherre.pgsql), [#15](https://www.postgresql.org/message-id/CAGRkXqTSqk-4ej3_q_frxGKvf-A8CcgCH_bvvEnvPuzaYe5u-w%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/1CE846D3-C504-436B-8F59-E9296CA22A4A%40gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Can ALTER INDEX ... ATTACH PARTITION avoid AccessExclusiveLock on ancestor tables and indexes when the attach validates no ancestor index?
2. How does the new AccessExclusiveLock on the root affect queries on hot standbys, where replay can make them wait or cancel them?
3. Which assertion, test or code rule makes a new caller of generate_partition_qual() fail loudly if it locks ancestors after the partition?
4. Should 0001 go ahead alone now, and will the general fix from message #16 keep the ancestor lock set it puts in place?
5. What happens when the target is reparented between the ancestor lookup and the lock on the target, which the patch only comments on?

## Touch points

- **WAL / replication / standby:** 0002 adds AccessExclusiveLock on ancestor tables and indexes, which Postgres WAL-logs and hot standbys replay.
- **ABI / extension API:** 0001 adds an exported function to src/include/catalog/partition.h.
- **backpatchability:** The entry is tagged Bugfix, but it changes the lock set of ATTACH PARTITION, which is a behaviour change for stable branches.
- **maintenance burden ("who maintains this in 10 years"):** Every path that walks from a partition up to its ancestors must pre-lock them, and nothing enforces this.
- **performance on the hot path:** Every ATTACH in a multi-level tree takes ancestor locks, and index ATTACH takes AccessExclusiveLock on the root table.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows partition locking in tablecmds.c and the index validation path. Message #14 asked for a committer decision on the index lock and got no reply.
- Priority: needs design rethink; audience niche; review cost medium. The deadlock needs a multi-level partition tree and concurrent DDL, and the author calls it rare and retryable. The patches are small, but the index lock trade-off has no committer answer and the author is holding the patch for a general fix.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
