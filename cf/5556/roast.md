# Design pre-review: Expanding HOT updates for expression and partial indexes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5556](https://commitfest.postgresql.org/patch/5556/); [thread](https://www.postgresql.org/message-id/flat/78574B24-BE0A-42C5-8075-3FA9FA63B8FC%40amazon.com), 88 of 88 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal, and demand is shown. Three fixable points remain. demand-1: updates that change an expensive index expression get slower, and the v46 messages name no opt-out. replication-ondisk-1: HOT safety now depends on extension answers that nothing checks. prior-attempts-1: the thread does not state the root cause of the 2018 revert. None of these stops review, but a committer who knows heapam and the index AMs must judge them.

Not checked: No code and no v46 patch prose were read. Whether the executor-side check keeps HOT, lock mode and replica identity decisions identical to master, and how toasted values are compared, could not be judged.

## Prior attempts

None found before 2025-02-06.

## Objections a reviewer is likely to raise

### 1. Users of costly expression indexes pay with no opt-out shown

A site keeps a to_tsvector() expression index on a text column that changes on most updates. After upgrade, each UPDATE evaluates the expression an extra time for the old row and gets no HOT update in return. The site sees the slowdown in production and finds no reloption to turn the check off.

Demand is real: JSONB expression indexes, ORM and logical replication write-back, a reverted earlier attempt, and reviewer support (messages #2, #45). The cost lands elsewhere. Message #81 reports +33-44% for changing expensive expressions, and that figure assumed 0010 removed one evaluation. Messages #83-84 dropped 0010, so v46 may be back to the 55% of message #79. Laurenz Albe asked for a per-table switch (message #2), the early versions had one, and the v46 messages do not mention it.

**Question:** What does v46 cost for an UPDATE that changes an expensive index expression, and can a table with such indexes opt out of the extra evaluation?

<sub>fixable, level 1b, performance on the hot path; messages [#2](https://www.postgresql.org/message-id/0a47ce53fa635787d36325b5ad7bac94cdca4246.camel%40cybertec.at), [#79](https://www.postgresql.org/message-id/-ugooulGn7M5X4DFIzWHSgYRngUmXChD_a-JvcMaklnDNcHMGnQhcM82SpYY76Je5ss4TOrsQB5sv2USLRSlDcpwDX-6CgCCbGKlJuA0SOs%3D%40protonmail.com), [#81](https://www.postgresql.org/message-id/j-s6ilS5lyXpV_6l6Q10JTSiumDI5TBUf2zsOfUZAsB3-TjQCJgxWWVIuoCvDZrhD5ljcouHg1s_S_FfjjBS-97Min2Wwyvm5vFSIcdsz_o%3D%40burd.me), [#83](https://www.postgresql.org/message-id/XtuZX8xWYFYMwfJ0BFIwteRjtst6uj1MVjGxiXNb8PuUJwkAUhvdQqeE1wbZPiYvmZ9G92o6CORDsp1vb3S8ci-lublEhHrFTil4GIYQBvU%3D%40burd.me), [#84](https://www.postgresql.org/message-id/atpXQaH1UBS3LY-6%40floki), [#88](https://www.postgresql.org/message-id/179434258467.3460405.2730116922465377144%40burd.me); `demand-1`</sub>

### 2. HOT chain correctness now depends on extension answers, with no check

An extension's SupportRequestSubattr handler or amcomparedatums (for example the RUM fork in message #79) wrongly reports an index's entries as unchanged. The UPDATE goes HOT and no new index entry is written. The primary and every physical standby then hold an index that misses the new key, and queries silently skip the row until someone runs REINDEX.

Today HOT safety rests on a byte comparison inside core. With 0005 and 0006 it also rests on proofs and comparisons written in extension code, and a wrong answer is stored on disk and replicated, not just a bad plan for one query. Message #81 checks proofs only for in-tree types ("misc_functions also checks HOT counts"). The brief lists what happens when an extension answers wrongly as unclear. Wrong answers from planner support functions affect one query, while this one persists in the index.

Precedent: [Revert changes in HOT handling of BRIN indexes](https://www.postgresql.org/message-id/flat/05ebcb44-f383-86e3-4f31-0a97a55634cf@enterprisedb.com) (Reverted, 2022-06-16)

**Question:** What stops, or detects, an index left without entries when an extension's SupportRequestSubattr or amcomparedatums answer is wrong, for example a debug check or an amcheck test?

<sub>fixable, level 2, ABI / extension API; messages [#79](https://www.postgresql.org/message-id/-ugooulGn7M5X4DFIzWHSgYRngUmXChD_a-JvcMaklnDNcHMGnQhcM82SpYY76Je5ss4TOrsQB5sv2USLRSlDcpwDX-6CgCCbGKlJuA0SOs%3D%40protonmail.com), [#81](https://www.postgresql.org/message-id/j-s6ilS5lyXpV_6l6Q10JTSiumDI5TBUf2zsOfUZAsB3-TjQCJgxWWVIuoCvDZrhD5ljcouHg1s_S_FfjjBS-97Min2Wwyvm5vFSIcdsz_o%3D%40burd.me), [#88](https://www.postgresql.org/message-id/179434258467.3460405.2730116922465377144%40burd.me); `replication-ondisk-1`</sub>

### 3. Design does not say why the 2018 surjective-index revert cannot recur

A committer pushes the series, and a user with an expression or GiST index hits a bug of the same family that forced the 2018 revert. Users see it as a crash or a wrong index result after release, and the only fix in reach is a revert of the series.

Message #1 says this grew out of surjective functional indexes, which were committed and then reverted for a bug and other quality issues. The only later trace of that bug is message #79 adding "Tom's 2018 gist point->box crash case" as a test. No message says which invariant that commit broke or why the executor-side check keeps it. Message #76 notes that a problem found after commit could only be met with a revert, which is how the earlier attempt ended.

**Question:** What was the root cause of the 2018 revert, and which part of the v46 design rules it out, beyond one regression test?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/78574B24-BE0A-42C5-8075-3FA9FA63B8FC%40amazon.com), [#76](https://www.postgresql.org/message-id/acQYVfzHoL4lVTUE%40nathan), [#79](https://www.postgresql.org/message-id/-ugooulGn7M5X4DFIzWHSgYRngUmXChD_a-JvcMaklnDNcHMGnQhcM82SpYY76Je5ss4TOrsQB5sv2USLRSlDcpwDX-6CgCCbGKlJuA0SOs%3D%40protonmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. What does v46 cost for an UPDATE that changes an expensive index expression, now that v42 dropped the patch that removed one evaluation?
2. Does v46 keep a per-table way, such as the expression_checks reloption, to skip the extra expression evaluation?
3. What was the root cause of the 2018 revert of surjective functional indexes, and which part of the v46 design rules it out?
4. What detects a wrong SupportRequestSubattr or amcomparedatums answer from an extension before the index is left without an entry?
5. How does v46 handle exclusion constraints, which 0009 of v41 covered and v42 dropped?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-1: HOT chain correctness now depends on extension answers, with no check
- **WAL / replication / standby:** see objection replication-ondisk-1: HOT chain correctness now depends on extension answers, with no check
- **ABI / extension API:** The series adds the amcomparedatums index AM callback and SupportRequestSubattr, and changes the table_tuple_update(), heap_update() and ExecInsertIndexTuples() signatures.
- **new GUC:** Early versions added the GUC enable_expression_checks, then the reloption expression_checks; the v46 messages do not say whether either remains.
- **psql UX conventions:** v15 touches psql tab completion for the expression_checks reloption.
- **maintenance burden ("who maintains this in 10 years"):** Nine patches change how the executor and heapam decide HOT, and add two contracts that extensions must answer correctly.
- **performance on the hot path:** Every UPDATE on a table with expression or partial indexes may evaluate the expression on the old row; message #81 reports 33-44% slower updates.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows heapam HOT rules, the executor update path and the index AM interface. Committers reviewed earlier versions, but the CommitFest entry has no reviewer now and v46 has no visible review.
- Priority: needs champion; audience a sizeable minority; review cost high. JSONB expression-index users, ORM write-back and extensions such as DocumentDB gain, and the author's numbers show large wins. The nine-patch series touches heapam, the executor and the index AM interface, and no qualified reviewer has looked at v46.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
