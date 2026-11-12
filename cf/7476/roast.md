# Design pre-review: Fix plpgsql's R/W optimization for simple expressions run via SPI

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7476](https://commitfest.postgresql.org/patch/7476/); [thread](https://www.postgresql.org/message-id/flat/CABUcsRRoJg_d%3D-%3DFGFRk2ZqMcDjXG_Yhpn9wuzfJXWa2AvnrpQ%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived at levels 1a, 1b or 2. Message #1 states the bug, gives a reproducer that returns a wrong result on 18 and master, and explains the cause. The fix narrows the read/write optimization to simple expressions and adds no user-visible surface. Confidence is low because only one message is visible and the code was not read.

Not checked: The code was not read, so the review could not check that the callback is installed only for the simple expression's own tree, or that the "inplace" method is safe on the SPI path. The memory-safety side of the second reference was also not checked.

## Prior attempts

None found before 2026-10-18.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does the "inplace" method, or any other user of expr_rwopt and expr_rw_param, have the same exposure when the expression runs through SPI after a replan?
2. How much slower is an assignment to a large array when the expression stays busy for the rest of a transaction after a caught error?
3. Do the TAP tests pass on master, 19 and 18 with the patch applied?

## Touch points

- **backpatchability:** The author asks for a back-patch to 18 and 19, and the CommitFest entry targets stable branches.
- **maintenance burden ("who maintains this in 10 years"):** The patch changes when pl_exec.c installs plpgsql_param_eval_var_check(), a rule that future changes to the read/write optimization must keep.
- **performance on the hot path:** Simple expressions run through SPI now copy the variable's value, where 18 could transfer it.

## Who would commit it, and when

- Owner: silent. Someone who knows PL/pgSQL expression execution in pl_exec.c, the expanded-object read/write rules, and how SPI replans cached plans.
- Priority: needs champion; audience niche; review cost low. The patch fixes a wrong result in released 18 that needs a busy simple expression and a replan, so few users hit it, but the result is silently wrong. The patch is 6.4 KB in one source file plus tests, and nobody has replied in the visible thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
