# Design pre-review: Add amnotifydelete, an index AM callback for deleted tuples

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7544](https://commitfest.postgresql.org/patch/7544/); [thread](https://www.postgresql.org/message-id/flat/CAA%2BRuFCYSbRr41E3Qt4cqKS%2B%2B_OQFiA%3DQUyidXL4f0c7zUiyKQ%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The proposal fails at level 1b on already-exists-1 and already-exists-2. It does not say what amnotifydelete does that an AFTER ROW trigger inserting markers cannot, or what core placement adds over an extension beyond the planner path. Demand shown is one workload, and message #1 says TimescaleDB works. Confidence is low because only the author has posted, so the verdict rests on missing answers.

Not checked: Without the code, nobody can check whether patch 1 covers every tuple-removing path or what the relcache check costs under concurrency. Message #4 was not shown, and the grain index type is not written.

## Prior attempts

- [Incremental Materialized View Maintenance](https://www.postgresql.org/message-id/flat/20181227215726.4d166b4874f8983a641123f5@sraoss.co.jp) (similar idea; Returned with feedback, 2022-10-12). "There has been some feedback sent lately and no update for one month, so I am marking it as RwF for now." ([message](https://www.postgresql.org/message-id/Y0ZyiXIEVa9i39ng%40paquier.xyz)) Different now: P1 kept aggregates current with AFTER triggers on a materialized view. This proposal keeps them in an index-owned summary table, using an executor callback, markers and a sealer watermark, so writers do not lock a shared rollup row.

## Objections a reviewer is likely to raise

### 1. Row-level triggers already deliver old values within the deleting transaction

A committer accepts amnotifydelete, which runs on every DELETE, non-HOT UPDATE, MERGE, ON CONFLICT, apply and REPACK path. Postgres now has two per-row notifications of deleted tuples, AFTER ROW triggers and this callback, and every future write path must remember to call both. If a new path misses the callback, nothing fails loudly, and a grain index silently drifts from the table.

An AFTER ROW DELETE or UPDATE trigger already receives the old row in the deleting transaction, which is the need the commit message states. Message #1 rejects triggers by pointing to the IVM patch, which locks the view and errors under REPEATABLE READ. But grain's own protocol only inserts marker rows when the time key is before W, and a trigger can do that without locking any summary row. The proposal does not show what the callback can do that such a trigger cannot.

**Question:** Which step of the grain protocol cannot be done by a C row trigger that inserts markers, with the same WaitForLockers sealer, and why?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAA%2BRuFCYSbRr41E3Qt4cqKS%2B%2B_OQFiA%3DQUyidXL4f0c7zUiyKQ%40mail.gmail.com); `already-exists-1`</sub>

### 2. The rollup can be built as an extension; core placement needs a reason

The project adds a grain index type with index-owned hidden heaps, new pg_dump and pg_upgrade handling, and a planner path. Every later catalog, dump or upgrade change must then account for a new kind of relation ownership. Users who need rollups today can already use TimescaleDB continuous aggregates, which merge stored buckets with recent raw rows past a watermark.

Message #1 says TimescaleDB works for this and calls it too heavy for one rollup. That makes it a convenience gap, since the job can be done today. A summary table, a marker table, a trigger, a sealer run by a background worker, and a pg_grain_buckets() function all fit in an extension. The thread has no demand beyond the author's own workload and no reply from anyone else.

**Question:** What does core placement give grain that an extension built on triggers, a background worker and a set-returning function cannot have, apart from the planner path?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAA%2BRuFCYSbRr41E3Qt4cqKS%2B%2B_OQFiA%3DQUyidXL4f0c7zUiyKQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAA%2BRuFAnX%3Ds6aD0Oj7CeT93Uh0nUY5GBtKa-G6zRKpSCRpJt5w%40mail.gmail.com); `already-exists-2`</sub>

## Questions for the author

1. Which step of the grain protocol cannot be done by a C row trigger that inserts markers, with the same WaitForLockers sealer?
2. What does core placement give grain that an extension with triggers, a background worker and a set-returning function cannot have, apart from the planner path?
3. Would the answers to your questions 1 and 2 change the amnotifydelete signature or call contract, and if so, why should patch 1 go in first?
4. What is the write cost of a grain index on inserts, updates and deletes, including marker inserts for rows before W?
5. How does the sealer run every minute, and how long does its WaitForLockers call wait when a long write transaction is open?

## Touch points

- **on-disk format / pg_upgrade:** The planned grain index keeps watermark W in its metapage and owns hidden summary and marker tables that pg_upgrade must carry over.
- **WAL / replication / standby:** Patch 1 adds calls in the logical replication apply paths; question 5 leaves the metapage WAL logging open.
- **catalog changes:** The planned index-owned hidden tables are a new kind of relation ownership; patch 1 adds a relcache flag.
- **ABI / extension API:** Patch 1 adds the amnotifydelete member to IndexAmRoutine.
- **SQL-standard conflict or non-standard syntax:** Planned `create index ... using grain ... with (buckets = ...)` and pg_grain_buckets() add non-standard surface.
- **planner behaviour change:** Patch 3 plans a planner path for group by date_bin(...) over a grain index.
- **pg_dump / restore round-trip:** Question 3 in message #1 leaves open how pg_dump handles the hidden summary and marker tables.
- **security / privileges:** The summaries aggregate all rows matching the index predicate, with no stated RLS or column privilege check on reads.
- **maintenance burden ("who maintains this in 10 years"):** Every current and future tuple-removing path must call amnotifydelete, and only dummy_index_am tests it.
- **performance on the hot path:** Every DELETE and non-HOT UPDATE on every table pays one cached relcache check per row.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #1 says TimescaleDB works, and the protocol may fit in an extension built on triggers.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the index AM API and the executor's ModifyTable and logical apply paths. Nobody but the author has posted.
- Priority: fix design first; audience unclear; review cost high. The thread shows one workload and no replies, so the audience is not shown. The patch is about 51 KiB across the executor, relcache, repack and logical apply, and its only planned user still has five open design questions.

---

[roast-full.md](roast-full.md) has 5 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
