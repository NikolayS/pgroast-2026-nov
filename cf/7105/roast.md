# Design pre-review: Add tests for concurrent DML retry paths in logical replication apply

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7105](https://commitfest.postgresql.org/patch/7105/); [thread](https://www.postgresql.org/message-id/flat/CALj2ACV6ESpggn2Az%3DOdZBzVx7jiBG9O_EdKdgbk2chAJusC2w%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No objection to the tests survived: the coverage report shows the retry paths are untested, and injection points cost production builds nothing. The two surviving objections, maintenance-fit-1 and maintenance-fit-2, concern only the bundled slot cleanup in execReplication.c. It has no measured gain, and its safety argument in message #5 rests on heap buffer pins and is unconfirmed by any reviewer. Both are fixable, most directly by the split the author already offers.

Not checked: Without the code, the review could not check that neither retry path reads the slot after the buffer pin could be released, or that the test is stable. Messages #4 and #5 were seen only through the objections.

## Prior attempts

None found before 2026-04-27.

## Objections a reviewer is likely to raise

### 1. Unmeasured slot cleanup bundled with tests in rarely run apply code

A committer pushes v3 for its tests. The removed materialization later turns out to matter in some case, and an apply worker on a subscriber writes a row built from stale or wrong values. Users see this as silent divergence between publisher and subscriber, long after the commit.

The tests add coverage at no runtime cost, but the execReplication.c cleanup changes behaviour in the conflict path that the tests are meant to protect. Message #1 says the gain is not a "major performance issue in practice", and there are no numbers. In message #4 the only reviewer questioned the materialization change, and nobody has confirmed the reply in message #5. The author offers to split the patch in message #5, and nobody has answered.

**Question:** Can the tests be committed first, with the cleanup as a separate patch backed by numbers or a concrete reason beyond consistency?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CALj2ACV6ESpggn2Az%3DOdZBzVx7jiBG9O_EdKdgbk2chAJusC2w%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/OS9PR01MB121491200B47154926F826084F5EC2%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#5](https://www.postgresql.org/message-id/CALj2ACWHZuGSU2m-DkStZYCvv0WxWAzVSnt9w1BZbRzvx0D2xg%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Safety argument for dropping materialization assumes heap buffer slots

An extension table AM whose slots do not hold a buffer pin is the target of logical replication. After v3, RelationFindReplTupleSeq scans straight into outslot and skips the earlier materialization. The AM author finds wrong or invalid tuple data only when replicated updates misbehave in the field.

Message #5 justifies the change by the pin that TTSOpsBufferHeapTuple keeps and by clock-sweep behaviour, which are heap and buffer-manager facts. execReplication.c works through the table AM API, so the invariant has to hold for any AM slot type, and also when the caller's outslot type differs from the scan slot type. Before v3, ExecCopySlot hid that difference. The proposal does not say why the change is safe outside heap.

**Question:** Why does skipping materialization and scanning directly into outslot stay correct for any table AM slot type, including when outslot differs from the AM's own slot type?

<sub>fixable, level 2, ABI / extension API; messages [#5](https://www.postgresql.org/message-id/CALj2ACWHZuGSU2m-DkStZYCvv0WxWAzVSnt9w1BZbRzvx0D2xg%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Can the tests and injection points be posted as a first patch, with the slot cleanup as a separate second patch?
2. Why is materializing only once before return correct for a table AM whose slots do not hold a buffer pin?
3. What measured cost or reported problem justifies the materialization change beyond consistency between the two functions?
4. How many injection points does v3 add, and what are their names, given that the commit message says one and message #3 says several?
5. How does the TAP test wait for the paused apply worker, and has it run repeatedly without failures on slow machines?

## Touch points

- **WAL / replication / standby:** The patch changes slot handling in the apply worker's tuple lookup on the subscriber.
- **ABI / extension API:** The lookup functions work through the table AM API, and the new materialization timing applies to any table AM's slots.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds injection points in execReplication.c and a concurrency TAP test that must stay stable on the buildfarm.
- **performance on the hot path:** RelationFindReplTupleSeq and RelationFindReplTupleByIndex run for every replicated update and delete; the patch removes a slot copy there.

## Who would commit it, and when

- Owner: silent. A committer who knows the logical replication apply code and tuple table slot lifetimes across table AMs would have to own it; none takes part in the visible thread.
- Priority: proceed; audience niche; review cost low. The patch is small (about 14 KiB, one backend file and one test) and serves developers of logical replication. The tests can go ahead once split from the cleanup, which needs its own justification.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
