# Design pre-review: skip check constraints verification for unaffected columns during UPDATE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6270](https://commitfest.postgresql.org/patch/6270/); [thread](https://www.postgresql.org/message-id/flat/CACJufxEtY1hdLcx%3DFhnqp-ERcV1PhbvELG5COy_CZjoEW76ZPQ%40mail.gmail.com), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The decisive objection is demand-1: the proposal is a performance optimization, and the thread shows no benchmark, workload or user report. The patch adds a new rule on which CHECK enforcement depends, so a committer would likely ask for numbers before reading the design. Confidence is low because the verdict rests on absent evidence and only 3 of 12 messages are visible. The level 2 objections security-1 and maintenance-fit-1 survived but are not evaluated here.

Not checked: Without the code, this review cannot confirm how v8 builds the updated-column set on each path, or whether it treats non-immutable CHECK expressions specially. Messages #4 to #12 were not visible, so some of the questions may already have answers there.

## Prior attempts

None found before 2025-12-01.

## Objections a reviewer is likely to raise

### 1. No measurement shows the skip saves meaningful time

A committer pushes the patch, and from then on every user relies on new skip logic to keep CHECK constraints enforced on UPDATE. Most CHECK expressions are cheap comparisons, so typical users may see no measurable gain. A future bug in the skip rules would let invalid rows into tables, and nobody would notice until queries or applications hit bad data.

The only demand shown is a Todo wiki item and an older thread (message #1). No user report, workload or benchmark appears in twelve messages, and the reviews in messages #9 and #12 check correctness, not speed. The change adds a correctness-critical path for constraint enforcement, with MERGE, ON CONFLICT and replication cases, so a committer would likely ask what users gain for that risk.

**Question:** Which constraint shapes and UPDATE workloads get a measurable speedup, and how large is it compared with the cost of the UPDATE itself?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CACJufxEtY1hdLcx%3DFhnqp-ERcV1PhbvELG5COy_CZjoEW76ZPQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CA%2BHEvJBziL%2BYmbs3zY4X6GNxdV3vdSw2djW9GSMRWVvV2nr2%3DQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAF0whueiCbq%3Dpistq0a6QGZrrYF_q3sm4Ti7%2Bk41xs6rRCRg6w%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which constraint shapes and UPDATE workloads get a measurable speedup from the skip, and how large is it compared with the cost of the UPDATE itself?
2. Does v8 skip a CHECK whose expression is not immutable, such as `check (owner = current_user)`, and if so, how are users told?
3. Does logical replication apply get the skip in v8, and does ExecSimpleRelationUpdate still pass CMD_INSERT to ExecConstraints?
4. What fails loudly when a future caller passes CMD_UPDATE with an incomplete updated-column set?
5. Does v8 keep the separate INSERT and UPDATE cached arrays from v7, and how does it cover INSERT ... ON CONFLICT DO UPDATE?

## Touch points

- **WAL / replication / standby:** The patch touches execReplication.c, and the thread does not say whether logical replication apply gets the skip.
- **ABI / extension API:** The patch changes the ResultRelInfo check-expression cache in execnodes.h and declarations in executor.h.
- **security / privileges:** A skipped CHECK such as `check (owner = current_user)` changes which role can update a row.
- **maintenance burden ("who maintains this in 10 years"):** CHECK enforcement on UPDATE now depends on the command type and a complete updated-column set in every caller.
- **performance on the hot path:** The skip decision sits in the constraint check that every UPDATE on a table with CHECK constraints runs.

## Who would commit it, and when

- Owner: silent. Someone who knows the executor's ModifyTable and constraint code would have to own it. Tom Lane asked one question in message #2, and the brief reports no committer reply to later versions.
- Not now, show demand first: A benchmark that shows a measurable UPDATE speedup for a named constraint shape, such as many or costly CHECK constraints on a wide table, compared with total UPDATE cost. A user report of CHECK re-evaluation as a bottleneck would also help.
- Priority: park; audience unclear; review cost medium. The idea follows the existing generated-column skip pattern and no surviving objection is fatal, but the thread shows no number for the gain. The patch is small (about 15 KB) but touches constraint enforcement in the executor, so review should wait for a benchmark.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
