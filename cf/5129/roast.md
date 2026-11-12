# Design pre-review: Asynchronous MergeAppend Execution

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5129](https://commitfest.postgresql.org/patch/5129/); [thread](https://www.postgresql.org/message-id/flat/59be194c5a409fb9fc9f2031581b8a44%40postgrespro.ru), 64 of 64 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and message #4 reports demand from production. Two fixable design points remain at level 2. maintenance-fit-1: an FDW that casts the ForeignAsyncConfigureWait requestor to AppendState compiles unchanged and fails only at run time. standards-compat-2: v25 does not say which setting turns off async MergeAppend. A committer who knows the async executor code is engaged and can settle both.

Not checked: The v25 code and its benchmarks were not seen. Without it, nobody can judge whether async heap filling keeps the merge order and cleans up pending requests on error; the withdrawn ordered Append side patch returned wrong results.

## Prior attempts

None found before 2024-07-17.

## Objections a reviewer is likely to raise

### 1. Async FDWs now receive MergeAppendState with no opt-in or loud failure

An out-of-tree FDW that is async-capable casts the ForeignAsyncConfigureWait requestor to AppendState, as the API has allowed until now. After upgrading, a query plans Merge Append over its scans and the FDW reads MergeAppendState fields as Append fields. The result can be wrong waits, a hang, or a crash, and nothing fails at compile time or at load time.

Message #54 states that the patch changes the current semantics of ForeignAsyncConfigureWait for out-of-tree FDWs. The answer in message #55 was to update fdwhandler.sgml. Documentation does not stop an existing binary from receiving a node type it never expected, and the default GUC is on, so the new path is taken without any action by the FDW author. A committer would likely ask whether async MergeAppend should apply only to FDWs that say they support it.

**Question:** How does an existing async-capable FDW that assumes an AppendState requestor avoid being used under Merge Append until its author updates it?

<sub>fixable, level 2, ABI / extension API; messages [#54](https://www.postgresql.org/message-id/a44e9ead6205f575da3df01b8c964e73%40postgrespro.ru), [#55](https://www.postgresql.org/message-id/CAPmGK15bYO_A-fTcBQnmjicAn%3DNPyOUVygDqjByXS%3Dga16p4Qw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. The GUC that controls async MergeAppend is not settled in v25

A site runs with enable_async_append = off to avoid async foreign scans. After upgrade, ordered queries switch to async MergeAppend because that path is gated by a separate enable_async_merge_append GUC that defaults to on, or by no GUC at all. The admin finds out from new wait events or a regression in production.

The early patches add enable_async_merge_append next to the existing enable_async_append (v1 config.sgml hunk). Message #36 still mentions it, but the v25 rewrite in messages #53 and #57 does not say whether it stays. A second enable_* setting for one feature departs from how enable_async_append is used today. If the GUC is dropped, the patch also does not say whether enable_async_append covers MergeAppend.

**Question:** In v25, which setting turns off async MergeAppend, and does enable_async_append = off still disable all async foreign scans?

<sub>fixable, level 2, new GUC; messages [#1](https://www.postgresql.org/message-id/59be194c5a409fb9fc9f2031581b8a44%40postgrespro.ru), [#36](https://www.postgresql.org/message-id/fa7ae004ec282fde84c29f36e0c85979%40postgrespro.ru), [#53](https://www.postgresql.org/message-id/CAPmGK15%3DKctsePnvKMNoYm3JOpuiZRWfWEfMs1GaBk9Rj45c5w%40mail.gmail.com), [#57](https://www.postgresql.org/message-id/CAPmGK14YATtpoUk6X54u1sW97mj-tccf4oc0hMX%3DmzG5mz02VQ%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. In v25, which setting turns off async MergeAppend, and does enable_async_append = off still disable all async foreign scans?
2. How does an async-capable FDW that casts the ForeignAsyncConfigureWait requestor to AppendState fail loudly, or decline use under Merge Append, until its author updates it?
3. What does v25 itself gain on the motivating case, with 20 or more postgres_fdw partitions and slow remote servers?
4. What does v25 cost a MergeAppend plan that has no async-capable children, compared with master?
5. Does v25 change the cost model, and if so, when does the planner now prefer MergeAppend over async Append?

## Touch points

- **ABI / extension API:** ForeignAsyncConfigureWait may now get a MergeAppendState as requestor, and execnodes.h changes.
- **new GUC:** Early versions add enable_async_merge_append; the material does not say if v25 keeps it.
- **planner behaviour change:** createplan.c and costsize.c change, and Foreign Scan children of MergeAppend become async by default.
- **maintenance burden ("who maintains this in 10 years"):** MergeAppend gets its own async code path next to Append's, plus a new wait event and FDW API text.
- **performance on the hot path:** nodeMergeAppend.c changes, and no v25 benchmark shows the cost for MergeAppend plans without async children.

## Who would commit it, and when

- Owner: qualified committer engaged. Etsuro Fujita reviews the entry, wrote v25, and plans to push it; the owner must know the async Append executor code and postgres_fdw.
- Priority: proceed; audience niche; review cost medium. The gain goes to users of ordered queries over postgres_fdw partitions, and message #4 reports a production slowdown there. v25 dropped the AppendBase refactoring and a committer is engaged; the v25 patch was not read, so review cost is an estimate.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
