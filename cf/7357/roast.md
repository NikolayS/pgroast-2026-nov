# Design pre-review: postgres_fdw: Fix costing of remote sorts without remote estimates

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7357](https://commitfest.postgresql.org/patch/7357/); [thread](https://www.postgresql.org/message-id/flat/DLRIUVXIEUOQ.2E2I0B7VZLHLI%40jeltef.nl), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). One objection survived, standards-compat-1, and it is a fixable level 2 point. Message #1 lists merge joins and merge appends turning into hash plans, and the thread shows no runtime numbers for those changes. The problem is stated and a reviewer accepts it, so levels 1a and 1b pass. The remaining work is measurements and a planner committer's judgment on new default plan shapes.

Not checked: The code was not read, so the claims that 0001 changes no plans and that the sort charge applies to remote work only are unchecked. Messages #4 to #8 were not shown in full, so the v3 position on aggregate costing is judged from the commit message alone.

## Prior attempts

None found before 2026-09-29.

## Objections a reviewer is likely to raise

### 1. Plan changes for existing postgres_fdw users, with no runtime evidence

A site queries a large remote table without use_remote_estimate, and the remote side has an index on the join key. Today it gets a merge join over remotely ordered scans. After upgrade the sort charge grows as N log N, the plan flips to a hash join that builds a large local hash table, and the site finds out when query latency changes.

The regression output already shows merge joins and merge appends turning into hash plans (message #1), so the patch removes remote ordering in some places. The author's "more pushdown is always better, no need for benchmarks" (message #3) does not cover those flips. Message #2 asked for TPC-H-style runs and the remote-index case, and nobody has answered with numbers. The only way back is use_remote_estimate, which costs an extra remote round trip per path. The 0.8 fraction is fixed, while fdw_startup_cost and fdw_tuple_cost are server options.

**Question:** On a large remote table with an index on the sort key, how do runtimes compare when the plan changes from a remote-sorted merge plan to a hash plan?

<sub>fixable, level 2, planner behaviour change; messages [#1](https://www.postgresql.org/message-id/DLRIUVXIEUOQ.2E2I0B7VZLHLI%40jeltef.nl), [#2](https://www.postgresql.org/message-id/CAFjuD9dj8tqqyz1NMOFU0Kr190sWz8qNbVAQApWD9p7YY9A4Wg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAGECzQSxLNtEuWWrU1K%3D2CAiGMRSV%2BUetK_nFz7%3DpdDKnHhUdw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. On a large remote table with an index on the sort key, how do runtimes compare when a remote-sorted merge plan becomes a hash plan?
2. How often do the new plans without use_remote_estimate match the plans chosen with use_remote_estimate = true, compared with the old costing?
3. Over what range of fractions did the plans stay the same, and at what table sizes was that tested?
4. Does the series remove DEFAULT_FDW_SORT_MULTIPLIER and the 1.05 special case, or do they remain in some code path?
5. Is the startup-cost fix in 0001 meant for back-patching, separately from the patches that change plans?

## Touch points

- **planner behaviour change:** The patch changes the cost of every sorted foreign path when use_remote_estimate is off, so existing plans change.
- **backpatchability:** The title says "Fix" and 0001 fixes a startup-cost bug, but the plan changes make the series unsuitable for stable branches.
- **maintenance burden ("who maintains this in 10 years"):** The patch restructures estimate_path_cost_size() and adds a fixed 0.8 fraction that someone must defend later.

## Who would commit it, and when

- Owner: silent. Someone who knows postgres_fdw path costing and the planner's sort and aggregate cost model would have to own it. No committer has posted in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost medium. The change affects postgres_fdw users on the default use_remote_estimate = false and adds no user-visible surface. Five patches in one contrib module are moderate to review, but the plan changes need runtime numbers and a planner-aware committer.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
