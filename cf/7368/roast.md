# Design pre-review: Two fixes for parallel query cleanup: DSM detach order and a stray statement timeout

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7368](https://commitfest.postgresql.org/patch/7368/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqQ7d%2BB%2BXDCNR0oUmG55EJ421HPj3saWONU2Uc0VKqARWg%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived the panel, so nothing stops either patch at levels 1a, 1b or 2. Both are bug fixes with measured reproducers, and 0002 restores the behaviour before 22f6f2c1ccb. Confidence is low because only message #1 is visible, no one has replied, and two safety points need the code.

Not checked: Without the code, the panel could not check whether 0002 clears only the timeout's own cancel request, or whether 0001 keeps the leader safe when a worker never exits. The pass at levels 1 and 2 is provisional on both points.

## Prior attempts

None found before 2026-09-30.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Can 0002 clear a cancel request from a client cancel or pg_cancel_backend() that arrives in the same window as the fired statement timeout?
2. With 0001, what does the leader do with the DSM segment if a worker never exits or the postmaster dies during WaitForParallelWorkersToExit()?
3. Do lock_timeout, transaction_timeout or the idle timeouts have the same fired-but-unserviced case, and does 0002 cover them?
4. Which back branches is each patch meant for, given that 0002 is described as a regression in 13 and later?
5. How does the attached reproducer set up the timing, so that a reviewer can repeat the 5 in 120 and 4 in 242 failure counts?

## Touch points

- **backpatchability:** Message #1 calls 0002 a regression in 13 and later but does not say which branches either patch targets.
- **maintenance burden ("who maintains this in 10 years"):** Both patches change cleanup ordering and interrupt bookkeeping in parallel.c and postgres.c, where timing bugs are hard to reproduce.
- **performance on the hot path:** 0002 changes disable_statement_timeout(), which runs once per statement; no added cost is claimed or shown.

## Who would commit it, and when

- Owner: silent. 0001 needs someone who knows parallel context shutdown and DSM lifetime. 0002 needs someone who knows the timeout and interrupt handling in postgres.c.
- Priority: proceed; audience a sizeable minority; review cost medium. 0002 fixes a case where a COMMIT fails and leaves an aborted transaction block, for users who combine statement_timeout with parallel queries. Both patches are small and add no user-visible surface, but the timing reasoning needs a reviewer who knows these two areas.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
