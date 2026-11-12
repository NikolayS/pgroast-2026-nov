# Design pre-review: Fix cumulative WAL statistics double-counting from parallel workers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7553](https://commitfest.postgresql.org/patch/7553/); [thread](https://www.postgresql.org/message-id/flat/CAP53PkzL-jF-Pm%2Bev2vBDvBYqDFNhcN4bpjGbGGbBfM_gtSdag%40mail.gmail.com), 24 of 24 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real and reproduced in message #1 and message #22, and no objection survived at level 1a or 1b. The surviving objections are fixable design points at level 2: maintenance-fit-1 (nothing enforces the worker subtraction), maintenance-fit-2 (whether master takes the same exported symbols as back branches) and standards-compat-1 (the per-backend attribution change in minor releases). Michael Paquier's objection for HEAD is still open in message #21, so a committer who knows the area must decide the shape per branch.

Not checked: Only 3 of 24 messages and no code were read, and v2 is not posted. The review could not check whether the new globals are static, how the pgstat_backend.c hunk behaves, or whether every InstrAccumParallelQuery caller is covered.

## Prior attempts

- [Parallel leader process info in EXPLAIN output](https://www.postgresql.org/message-id/flat/CA+hUKG+Z22=vkVkXtKRznzRdtj=MtygfZMYUJs8j7ObjzkG1Lw@mail.gmail.com) (related; Returned with feedback, 2020-03-17). "I think this is doubling down on a fundamentally wrong decision to have ExecParallelRetrieveInstrumentation do some aggregation immediately." ([message](https://www.postgresql.org/message-id/18323.1580078950%40sss.pgh.pa.us)) Different now: P7 added leader-only EXPLAIN output and a PlanState field, not a cumulative stats fix. It shares only the background that worker data is merged into leader state early.
- [Revert "Fix accumulation of parallel worker instrumentation."](https://www.postgresql.org/message-id/flat/CAA4eK1LW2aFKzY3=vwvc=t-juzPPVWP2uT1bpx_MeyEqnM+p8g@mail.gmail.com) (related; Reverted, 2017-12-13). "Per further discussion, that doesn't seem to be the best possible fix." Different now: P3 reverted a fix to how parallel worker executor instrumentation was accumulated, not WAL cumulative stats. The row gives no reason that applies to this patch.

## Objections a reviewer is likely to raise

### 1. Unenforced rule: every pgWalUsage diff for cumulative stats must subtract workers

A later patch diffs pgBufferUsage to feed pg_stat_database, or adds another WAL-based stats kind, and uses the global directly instead of a ComputeOwn*Usage helper. Parallel worker activity is counted twice again. Nothing fails at build or test time, and users find out from inflated monitoring numbers years later, as happened with pg_stat_wal since version 14.

Message #3 says nothing guarantees the invariant, and the planned guard is a comment in instrument.h (messages #3, #7). The patch adds no regression test, and reviewers chose not to ask for one (message #7). The author already plans the next user of this pattern, buffer timing for pg_stat_database (message #1), so the same trap gets a second consumer soon.

**Question:** What makes a future patch that diffs pgWalUsage or pgBufferUsage for cumulative stats, without the worker subtraction, fail in a test or at review instead of in the field?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAP53PkzL-jF-Pm%2Bev2vBDvBYqDFNhcN4bpjGbGGbBfM_gtSdag%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAP53PkxYDUEa_cjRT9HWfzFrvQqYMfLngaEh7kWYkwLABTVfPw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/auhKz%2FRirAnqHl3f%40bdtpg); `maintenance-fit-1`</sub>

### 2. Master gets exported API the author calls an interim workaround

Master ships ComputeOwnWalUsage() and pgWalUsageFromWorkers/pgBufferUsageFromWorkers, and an extension starts using them. Stack-based instrumentation then removes or redefines them one release later, and the extension breaks. Meanwhile pgBufferUsageFromWorkers is accumulated on every parallel query but no stats consumer reads it yet.

The author calls approach (1) "ugly" and a step before a more elegant solution in the stack-based thread (message #20). Michael objects to it on HEAD as an abuse of pgWalUsage (messages #13, #21), and Andres says the stack-based work removes the main problem with approach (2) (message #17). Back branches may need approach (1), but master could take a leader-only design or a fix with no new exported symbols. [P3] reverted a parallel instrumentation fix that "doesn't seem to be the best possible fix".

Precedent: [Revert "Fix accumulation of parallel worker instrumentation."](https://www.postgresql.org/message-id/flat/CAA4eK1LW2aFKzY3=vwvc=t-juzPPVWP2uT1bpx_MeyEqnM+p8g@mail.gmail.com) (Reverted, 2017-12-13)

**Question:** Why should master take the same exported globals and helper as the back branches, instead of a static-only fix or the leader-only design that the stack-based work allows?

<sub>fixable, level 2, ABI / extension API; messages [#13](https://www.postgresql.org/message-id/aurfeqkq2dgqvdZK%40paquier.xyz), [#17](https://www.postgresql.org/message-id/cz3lbv54b477owlxu73vtyfsoomg3snvpgdb5jtiqogffeznpa%40x2ecg363asoc), [#20](https://www.postgresql.org/message-id/CAP53Pkz%3DvE7X9BkSbxGJExERwHLqRHTsqgZo0hD_uQdO3bEFKA%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/auwVNOik4oOhAIjv%40paquier.xyz); `maintenance-fit-2`</sub>

### 3. Backpatch removes parallel worker WAL from per-backend stats with no replacement

A monitoring tool samples pg_stat_get_backend_wal() for a long-lived session that runs parallel VACUUM. After a minor release, that leader's wal_bytes drops by about a third in message #22's test. The worker WAL now shows up in no per-backend entry at all, because each worker's entry is dropped when it exits.

On stable branches, pg_stat_get_backend_wal() counts worker WAL once, in the leader's entry (message #22). The pgstat_backend.c hunk is an attribution change, not a double-counting fix, and message #23 and message #24 agree on that. It matches the documentation, but it moves WAL out of the only per-backend view that showed it, and message #24 leaves a per-leader worker view to future work. Fixing pg_stat_wal does not need this hunk, so a committer would likely ask why a reporting change goes into minor releases with the bug fix.

**Question:** Why should the pg_stat_get_backend_wal() attribution change go to stable branches rather than only to HEAD, where a per-leader worker view could come with it?

<sub>fixable, level 2, backpatchability; messages [#22](https://www.postgresql.org/message-id/CAGRkXqQskUo2s539Wk%2BiYav1mp%3DcDcMPT%2BofVXHFRE4HTRxfBQ%40mail.gmail.com), [#23](https://www.postgresql.org/message-id/avKaMOu4yUik38O2%40bdtpg), [#24](https://www.postgresql.org/message-id/CAP53Pkxo%3DTT3iOXxinGGrQEozt1Ai%2BNjdAy7uWrPkjCJK6LzRg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. What makes a future patch that diffs pgWalUsage or pgBufferUsage for cumulative stats, without the worker subtraction, fail in a test or at review?
2. Why should master take the same exported globals and helper as the back branches, instead of a static-only fix or the leader-only design?
3. Why should the pg_stat_get_backend_wal() attribution change go to stable branches, where no per-backend entry shows worker WAL afterwards?
4. Does the backpatch add exported symbols to stable branches, or will v2 make pgWalUsageFromWorkers and pgBufferUsageFromWorkers static?
5. Does any buffer-based cumulative statistic change with this patch, or is pgBufferUsageFromWorkers accumulated with no reader yet?

## Touch points

- **ABI / extension API:** instrument.h gains exported globals pgWalUsageFromWorkers and pgBufferUsageFromWorkers and the function ComputeOwnWalUsage(), also on stable branches.
- **backpatchability:** The author proposes a backpatch to all branches, which changes pg_stat_wal and pg_stat_get_backend_wal() values in minor releases.
- **maintenance burden ("who maintains this in 10 years"):** Every future cumulative-stats user of pgWalUsage or pgBufferUsage must subtract the worker part, and only a comment guards that rule.
- **performance on the hot path:** InstrAccumParallelQuery does extra additions per worker; the subtraction runs only at stats flush.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund and Michael Paquier take part in the thread but disagree on the shape for HEAD. The owner needs to know the cumulative stats code and parallel instrumentation.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 10 KiB in four files and fixes wrong pg_stat_wal totals for anyone running parallel vacuum or parallel scans. The open points are the shape on HEAD and the scope of the backpatch, which a v2 and the engaged committers can settle.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
