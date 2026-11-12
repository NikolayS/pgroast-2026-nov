# Design pre-review: A new C function `get_partition_root`.

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7023](https://commitfest.postgresql.org/patch/7023/); [thread](https://www.postgresql.org/message-id/flat/CAHut%2BPtc73UpWSn4teMyMxQoHwGJjRwhWiu%3DhGJqAj5k2AOXEw%40mail.gmail.com), 20 of 20 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal, and reviewers in messages #2 and #14 accept the helper. Two fixable points remain. maintenance-fit-1: with even_if_detached false the helper returns InvalidOid, while get_partition_parent() raises an error. demand-1: the callers that would settle that contract are in the unmerged EXCEPT patches. A committer who knows the partition detach code needs to decide the contract.

Not checked: The code was not read and only 3 of 20 messages were shown. Two points are unchecked: which even_if_detached value pg_partition_root() passes, and whether its result changes for a partition under a pending detach.

## Prior attempts

None found before 2026-07-17.

## Objections a reviewer is likely to raise

### 1. Main demand rests on unmerged EXCEPT patches; two callers today

A committer pushes get_partition_root() with its even_if_detached flag and InvalidOid-on-detach behaviour. The FOR TABLES IN SCHEMA EXCEPT patches then change shape or stall. Core is left with an exported helper whose detach semantics were never checked against the callers it was designed for.

In core, only pg_partition_root() and one site in pg_depend.c use the helper. Message #1 justifies it mainly by "many more code fragments" in the EXCEPT patches, which are "still in development". Message #2 also ties it to that patch. Messages #17 and #20 leave the detach behaviour open, and only real callers can settle whether an error or InvalidOid is the right answer.

**Question:** Which EXCEPT call sites would use get_partition_root(), and how does each one handle a pending detach, ideally shown as patch 0001 of that series?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAHut%2BPtc73UpWSn4teMyMxQoHwGJjRwhWiu%3DhGJqAj5k2AOXEw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAJpy0uDrs7ag3QwU7QgDNXOo9trLxgq881Qu1TgzuJVwsNpnHA%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAJpy0uDr_DzUvkSYwYrkuD7r3VOcu4VAU-BqtBPP5Xv-9iWQfA%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAJpy0uBWQ2hfCU2w%2B_JZ-_a2dbdw0msJ%3Dt1_ebieTf5wbKXDJw%40mail.gmail.com); `demand-1`</sub>

### 2. Detach-pending case returns InvalidOid, unlike get_partition_parent

An EXCEPT-patch caller writes relid = get_partition_root(relid, false) and passes the result to a syscache lookup or table_open. During an ALTER TABLE ... DETACH PARTITION CONCURRENTLY, it gets InvalidOid, and the failure shows up as an odd 'cache lookup failed for relation 0' or a silent mismatch. Nobody sees it until a user hits the race in production.

Chao Li approved even_if_detached because it matches get_partition_parent() (message #12). With false, get_partition_parent() raises an error, but the new helper returns InvalidOid. Message #4 shows a caller pattern, relid = get_partition_root(relid), that assigns the result without checking it. The thread lists this as open (message #20), and no committer has answered. A helper meant for many future callers should not need each caller to remember a sentinel check when its sibling function raises an error instead.

**Question:** Why should get_partition_root(relid, false) return InvalidOid instead of erroring like get_partition_parent(), and how will callers be kept from using the result unchecked?

<sub>fixable, level 2, ABI / extension API; messages [#4](https://www.postgresql.org/message-id/CAJpy0uD1Yxm54Y%3DDJsE2vL%2Bifws7V0HpMJBerJMLFU9B8YTxEQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/107B6F21-277C-4A71-AAC2-0E79918AFD5C%40gmail.com), [#17](https://www.postgresql.org/message-id/CAJpy0uDr_DzUvkSYwYrkuD7r3VOcu4VAU-BqtBPP5Xv-9iWQfA%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAJpy0uBWQ2hfCU2w%2B_JZ-_a2dbdw0msJ%3Dt1_ebieTf5wbKXDJw%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Why does get_partition_root(relid, false) return InvalidOid during a pending detach instead of raising an error like get_partition_parent()?
2. Which call sites in the FOR TABLES IN SCHEMA EXCEPT patches would use get_partition_root(), and what does each one do when a detach is pending?
3. Should input validation use if/elog instead of Assert, as Álvaro Herrera asked, given that the Assert checks do catalog lookups?
4. How does a caller tell a dropped relation from a relation that is not a partition?
5. Does the patch fix the crash risk that message #11 reports for existing get_partition_ancestors() callers during a detach, or is that out of scope?

## Touch points

- **ABI / extension API:** The patch exports get_partition_root() in catalog/partition.h and moves check_rel_can_be_partition() to shared code.
- **maintenance burden ("who maintains this in 10 years"):** Core gets a second way to find the root, and its pending-detach behaviour differs from get_partition_parent().

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera reviewed the patch and his objection led to v7, but the thread shows no reply from him on v7. The owner needs to know the DETACH PARTITION CONCURRENTLY code.
- Priority: bundle; audience niche; review cost low. The patch is about 9 KiB over four files and has two callers in core today. Its main users are the unmerged EXCEPT patches, so reviewing it as the first patch of that series would settle the detach contract against real callers.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
