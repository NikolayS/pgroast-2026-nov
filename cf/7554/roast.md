# Design pre-review: Assert failure in parallel query due to SubPlan double visit for hash join

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7554](https://commitfest.postgresql.org/patch/7554/); [thread](https://www.postgresql.org/message-id/flat/CAP53PkzYGGESHX%2Bdsy04rcO-7PpBjGg-QNzfLcAC%2BagydukTcA%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real: message #1 gives a reproducer and messages #2 and #3 confirm it. No objection survived at levels 1a or 1b. Two fixable design points remain at level 2: standards-compat-1 (no fix shape for the stable branches, where the bug has existed since Postgres 10) and maintenance-fit-2 (the Bitmapset-first context rule is not checked at compile time for out-of-core callers). Neither would stop the fix; both need a decision from someone who knows the executor.

Not checked: The code was not read, and the favored walker-level design has no posted patch. Messages #4 to #8 were not shown, so the context-struct details and the walker cost could not be checked.

## Prior attempts

None found before 2026-10-26.

## Objections a reviewer is likely to raise

### 1. Proposed walker change cannot reach the back branches that have the bug

A packager ships a minor release with the fix after it is back-patched to stable branches. An extension that calls planstate_tree_walker() or the changed execParallel.h functions, built against the earlier minor release, now passes a context of the wrong shape. It misbehaves or crashes, and users find out only after the minor upgrade.

The bug dates from Postgres 10, so it is present in every supported branch (message #1). v1 changes exported signatures in execParallel.h. The design now favored in messages #7 and #8 changes the context contract of planstate_tree_walker(), which is part of the extension API. Neither version says whether there will be a separate fix for the back branches that keeps the ABI stable.

**Question:** What fix ships to the supported back branches, and does it leave the signatures of planstate_tree_walker() and the execParallel.h functions unchanged?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAP53PkzYGGESHX%2Bdsy04rcO-7PpBjGg-QNzfLcAC%2BagydukTcA%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAP53PkwgWzwBbY8mYmeqGqMSKURD07H_vQbqjkdtcROttBpn9w%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/b0fa92db-7fc1-4452-9f5c-54fe721d331e%40gmail.com); `standards-compat-1`</sub>

### 2. Required Bitmapset-first context changes walker contract silently for extensions

An extension calls planstate_tree_walker() with its own context struct, which has no Bitmapset as its first member. The code still compiles against the new release. On a production build the walker reads the first field as a Bitmapset and crashes or skips nodes. The Bitmapset node-tag assertion only catches this on assert builds.

Message #8 proposes requiring "the first member of any context is a Bitmapset" behind the existing void * context. That is a convention the compiler cannot check. Message #8 also counts only in-core call sites, while planstate_tree_walker() is exported and used by out-of-core code. Option 2 in message #7, an explicit extra argument, breaks callers at compile time, and message #8 rejects it only for convenience at the call sites.

**Question:** How will out-of-core callers of planstate_tree_walker() learn about the new context requirement at compile time rather than through a crash on non-assert builds?

<sub>fixable, level 2, ABI / extension API; messages [#7](https://www.postgresql.org/message-id/CAP53PkwgWzwBbY8mYmeqGqMSKURD07H_vQbqjkdtcROttBpn9w%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/b0fa92db-7fc1-4452-9f5c-54fe721d331e%40gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Which supported branches get the fix, and what change goes there that leaves planstate_tree_walker() and the execParallel.h functions unchanged?
2. How does an out-of-core caller of planstate_tree_walker() learn at compile time that its context must start with a Bitmapset?
3. Why is the Bitmapset-first context preferred over an explicit extra argument, which would make old callers fail to compile?
4. What does the visited check cost per walk for callers that never meet a shared SubPlan?
5. Is a CustomScan shutdown callback that is not idempotent a real case that the stable-branch fix must also cover?

## Touch points

- **ABI / extension API:** v1 adds planstate_tree_collect() to nodeFuncs.h and changes exported functions in execParallel.h; the favored design changes the planstate_tree_walker() context contract.
- **backpatchability:** The bug dates from Postgres 10, and the thread does not say which fix goes to stable branches.
- **maintenance burden ("who maintains this in 10 years"):** Every future planstate_tree_walker() caller must follow the new visit-once rule or context layout.
- **performance on the hot path:** Dedup in planstate_tree_walker() adds a visited check to every PlanState walk; no one has measured it.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the executor's parallel query setup (execParallel.c) and the nodeFuncs.c walkers. The visible thread shows the author and two reviewers; the author of b8d7f053c5c was CCed and has not replied.
- Priority: proceed; audience niche; review cost medium. The plan shape is uncommon, but the assertion failure and the wrong EXPLAIN ANALYZE output are confirmed by two reviewers. The v1 patch is about 31 KiB in execParallel.c and nodeFuncs.c, and the walker contract change needs care for extension callers.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
