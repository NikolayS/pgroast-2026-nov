# Design pre-review: llvmjit: always add the simplifycfg pass

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6384](https://commitfest.postgresql.org/patch/6384/); [thread](https://www.postgresql.org/message-id/flat/VS3dpR1Sf5jGnWwoFFJ-_x3GbW7fdmV0arzWPIDfrmbVzewifgu6DsQ7oDa-TAwRz9N2p817j3jGstHwfPOJJxOipbcp-nHdNj3zyxKvC4Q%3D%40pinaraf.info), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection to the goal or to the design survived. The one surviving objection, maintenance-fit-1, is fixable: the thread has no compile-time numbers for deform functions with many columns, which message #7 asked for. The O0 path runs for every query above jit_above_cost, so someone who knows llvmjit has to judge that cost.

Not checked: Only 3 of 10 messages and no code were read. The pass order and the compile cost on large deform functions could not be checked.

## Prior attempts

- [JIT: Optimize generated functions earlier to lower memory usage](https://www.postgresql.org/message-id/flat/CAE-ML+_OE4-sHvn0AA_qakc5qkZvQvainxwb1ztuuT67SPMegw@mail.gmail.com) (related; Withdrawn, 2019-11-12). The visible messages do not show why. Only the author's opening message is shown, and the entry was withdrawn about a month later with no visible review or reply. (AI summary of the thread) Different now: P2 moved function-level passes earlier to lower memory use; this proposal adds one pass to the O0 pipeline for better code. P2 was withdrawn with no recorded reason.

## Objections a reviewer is likely to raise

### 1. Compile cost on wide deform functions is not measured

With jit = on by default, a query on a table with hundreds of columns crosses jit_above_cost but stays below jit_optimize_above_cost. Every execution then pays simplifycfg on a large deform function, the case where compile time matters most. A user notices it only as higher Optimization time in EXPLAIN ANALYZE.

The O0 path serves queries judged too cheap for full optimization, so extra compile time there is felt more than anywhere else. The only numbers are a small single-column query (message #1) and TPC-H Q1 and Q4 (message #4). In message #7, a committer who knows llvmjit said these do not exercise deform routines with many columns. No later message answers this.

**Question:** What do Optimization time and execution time look like for deform functions with about 100 to 1000 columns, with jit_optimize_above_cost = -1?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/VS3dpR1Sf5jGnWwoFFJ-_x3GbW7fdmV0arzWPIDfrmbVzewifgu6DsQ7oDa-TAwRz9N2p817j3jGstHwfPOJJxOipbcp-nHdNj3zyxKvC4Q%3D%40pinaraf.info), [#4](https://www.postgresql.org/message-id/DFVG0AV650GW.2CNS5CZ4OG788%40gmail.com), [#7](https://www.postgresql.org/message-id/porx6mjfalwefma2f2d76hagxjin3xdgefjsklkzxsyit736ly%4034ubivd7buw2); `maintenance-fit-1`</sub>

## Questions for the author

1. What are Optimization time and execution time for deform functions with about 100 to 1000 columns, with jit_optimize_above_cost = -1?
2. Why does simplifycfg remove the stack spills that mem2reg leaves in deform functions?
3. Should mem2reg run before or after simplifycfg, and does the order change compile time or the generated code?
4. Do the gains hold on LLVM versions other than 19 and on architectures other than x86_64?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch changes the LLVM pass strings in llvmjit.c, which must behave well across all supported LLVM versions.
- **performance on the hot path:** Every query compiled by JIT below jit_optimize_above_cost pays the extra simplifycfg pass at compile time.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund, who knows llvmjit, raised the benchmark question in the thread; a committer who knows the LLVM pass pipeline would have to own it.
- Priority: proceed; audience a sizeable minority; review cost low. The patch touches only llvmjit.c, adds no user-visible surface, and a reviewer measured TPC-H Q4 about 18% faster. The missing piece is one measurement on wide tables.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
