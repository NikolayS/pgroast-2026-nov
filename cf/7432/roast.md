# Design pre-review: Fix leak when a plpgsql exception block catches an error from CALL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7432](https://commitfest.postgresql.org/patch/7432/); [thread](https://www.postgresql.org/message-id/flat/CABUcsRSoj8i475YkPtVFETbaOvSzA4SnAwPRLr5bJKF3gieASQ%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at level 1a or 1b: message #1 gives a reproducible leak, its cause and measurements. The only surviving objection, maintenance-fit-1, is a small fixable design point at level 2. The fix changes which resource owner holds a plan pin and is meant for back-patch to 14 through 19, so a committer who knows the PL/pgSQL and SPI code has to sign for it. Nobody has replied yet.

Not checked: The code was not read. Not checked: whether every path that can end the transaction inside a CALL or DO is covered by SPI_inside_nonatomic_context(), and whether the new test is stable on the buildfarm.

## Prior attempts

None found before 2026-10-11.

## Objections a reviewer is likely to raise

### 1. Correctness depends on two copies of the non-atomic rule agreeing

A later patch changes when _SPI_execute_plan() runs a CALL non-atomically but leaves SPI_inside_nonatomic_context() unchanged. A CALL then runs non-atomically while its plan is pinned in the transaction's resource owner, and a COMMIT in the callee releases the pin while the plan is still in use. Users would hit this as a crash or a refcount warning on an uncommon path, long after the change was committed.

Message #1 says SPI_inside_nonatomic_context() "also implements" the rule from c96de42c4b5, which reads as a second copy of the rule that _SPI_execute_plan() applies, not a shared helper. After this patch, pl_exec.c relies on both copies giving the same answer to decide which resource owner holds the pin. The proposal does not say what fails loudly, such as an assertion, if the two copies drift apart. This is a small point for a back-patched bug fix, but a committer would likely ask for it before signing for the change on six branches.

**Question:** Do exec_stmt_call() and _SPI_execute_plan() share one test for non-atomic execution, or can an assertion catch a CALL that runs non-atomically while its pin is in the current resource owner?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CABUcsRSoj8i475YkPtVFETbaOvSzA4SnAwPRLr5bJKF3gieASQ%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Do exec_stmt_call() and _SPI_execute_plan() share one test for non-atomic execution, or can an assertion catch a CALL that runs non-atomically with its pin in the current resource owner?
2. Which existing test fails if SPI_inside_nonatomic_context() and _SPI_execute_plan() disagree about whether a CALL runs non-atomically?
3. Does the test that counts CachedPlan contexts in pg_backend_memory_contexts give stable output on all six branches under parallel schedules and unusual build options?
4. Do the TAP tests pass with the patch on master and on the back branches?
5. Does any other PL or extension depend on an atomic DO block pinning its CALL plans in the block's simple-expression resource owner?

## Touch points

- **backpatchability:** The author asks for a back-patch to 14 through 19 and adds a new regression test on each branch.
- **maintenance burden ("who maintains this in 10 years"):** exec_stmt_call() now depends on SPI_inside_nonatomic_context() agreeing with the non-atomic test in _SPI_execute_plan().
- **performance on the hot path:** The patch adds one test to every PL/pgSQL CALL; message #1 measures it as within noise.

## Who would commit it, and when

- Owner: silent. Someone who knows the PL/pgSQL executor, SPI plan execution and the plan cache's resource owner handling; none has replied in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is 7.8 KB, touches only pl_exec.c and the plpgsql_call test, and adds no user-visible surface. It fixes unbounded memory growth of about 2 kB per caught CALL failure in batch procedures.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
