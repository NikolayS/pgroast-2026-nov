# Design pre-review: Gather rescan keeps the first scan's tuple bound in workers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7537](https://commitfest.postgresql.org/patch/7537/); [thread](https://www.postgresql.org/message-id/flat/2701c83e33f3fe66320291fb5d1b74ca2cb90bd7.camel%40j-davis.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is a confirmed wrong-results bug with a default-settings repro in message #1, and the fix follows what ExecInitParallelPlan() already does. The only surviving objection is standards-compat-1: the patch says Backpatch-through: 14 and changes an exported function signature in back branches. That is a fixable choice, and message #2 already names the alternative. No level fails.

Not checked: Without the code, this review could not check that both Gather and Gather Merge pass the current bound at the right point on rescan. It also could not check whether the test output is deterministic.

## Prior attempts

None found before 2026-09-18.

## Objections a reviewer is likely to raise

### 1. Back-branch fix changes the signature of an exported executor function

An extension built against an earlier 14-18 minor release calls ExecParallelReinitialize() with the old argument list. After a minor upgrade the server reads an undefined tuples_needed value. The workers then stop at a random bound, or the extension fails to build, and the user finds out only after the upgrade.

Patch 0001 says "Backpatch-through: 14" and adds a parameter to a function declared in execParallel.h. Minor releases normally keep exported signatures stable, so an extension does not need a rebuild. In message #2 the author says no caller is known and offers a new function as the alternative, but the thread does not choose. An absence of known callers is weak evidence for out-of-tree code that uses parallel contexts.

**Question:** Will the back-branch versions keep the existing ExecParallelReinitialize() signature, for example by setting the bound through a separate function, and leave the new argument only on master?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/CAGRkXqRGE%2BTwD0Ttmb4b1J8Vy-drrkZYhh6hOf2_Uz5XKtXsEA%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Will the back-branch versions keep the existing ExecParallelReinitialize() signature, for example with a separate function that sets the bound, and change the signature only on master?
2. Is the 0002 regression test stable across platforms and repeated runs, given that it depends on which rows the workers read with the leader out of the scan?
3. Does the fix also cover the plain Gather case from message #1, and does the test exercise it or only Gather Merge?

## Touch points

- **ABI / extension API:** ExecParallelReinitialize() in execParallel.h gains a tuples_needed argument.
- **backpatchability:** Patch 0001 says Backpatch-through: 14 and changes an exported signature in those branches.

## Who would commit it, and when

- Owner: qualified committer engaged. Jeff Davis reported and confirmed the bug in message #1 but has not reviewed the patch. The owner needs to know the parallel executor's DSM reuse on rescan.
- Priority: proceed; audience a sizeable minority; review cost low. The query returns wrong results without an error at default settings in all supported branches, and the fix is about 4 KiB in three executor files and one header. The plan shape needs a parameter-dependent Limit over a rescanned Gather or Gather Merge, so not every user hits it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
