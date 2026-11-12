# Design pre-review: llvmjit: always add the simplifycfg pass

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6384](https://commitfest.postgresql.org/patch/6384/) (Needs review); authors: Pierre Ducroquet (pinaraf)
- Thread: [\[PATCH\] llvmjit: always add the simplifycfg pass](https://www.postgresql.org/message-id/flat/VS3dpR1Sf5jGnWwoFFJ-_x3GbW7fdmV0arzWPIDfrmbVzewifgu6DsQ7oDa-TAwRz9N2p817j3jGstHwfPOJJxOipbcp-nHdNj3zyxKvC4Q%3D%40pinaraf.info), started 2026-01-07
- Messages read: 10 of 10
- Patch prose read: 0001-llvmjit-always-use-the-simplifycfg-pass.patch, 0001-llvmjit-always-use-the-simplifycfg-pass.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection to the goal or to the design survived. The one surviving objection, maintenance-fit-1, is fixable: the thread has no compile-time numbers for deform functions with many columns, which message #7 asked for. The O0 path runs for every query above jit_above_cost, so someone who knows llvmjit has to judge that cost.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 10 messages and no code were read. The pass order and the compile cost on large deform functions could not be checked.

## The proposal, restated

**Problem.** At O0, llvmjit does not run the simplifycfg pass, so tuple deforming code keeps many empty basic blocks. The generated machine code contains chains of redundant jumps and stack spills. This affects queries that use JIT but stay below jit_optimize_above_cost.

**Proposed design.** The patch adds simplifycfg to the LLVM pass pipeline whenever default<O0> is the base. This covers both the plain path (default<O0>,mem2reg,simplifycfg) and the inlining path (default<O0>,mem2reg,simplifycfg,inline). The default<O3> pipeline is unchanged because it already includes simplifycfg.

**What the author claims.**
- On a simple select * from table where f = 42, the patch saved 7ms of runtime for 0.1ms of extra optimization time; the commit message says 0.2ms.
- Code from the llvmjit provider, especially tuple deforming, depends heavily on simplifycfg, so O0 pipelines should always include it.
- Other passes the author tried did not give a similar benefit for their cost.
- With simplifycfg, stack spills in deform functions go away, which suggests mem2reg alone does less than expected.

**Thread state.** Matheus Alcantara reviewed it, posted TPC-H Q1 and Q4 numbers (Q4 about 18% faster, Q1 unchanged) and gave +1. Andres Freund doubts these queries are a good benchmark, and the open question is why simplifycfg removes stack allocations that mem2reg did not.

## 2. Prior attempts

### JIT: Optimize generated functions earlier to lower memory usage (related)

- **Link:** [https://commitfest.postgresql.org/patch/2298/](https://commitfest.postgresql.org/patch/2298/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAE-ML+_OE4-sHvn0AA_qakc5qkZvQvainxwb1ztuuT67SPMegw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAE-ML+_OE4-sHvn0AA_qakc5qkZvQvainxwb1ztuuT67SPMegw@mail.gmail.com)
- **Date:** decided 2019-11-12; started 2019-10-07
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. Only the author's opening message is shown, and the entry was withdrawn about a month later with no visible review or reply.
- **Reason (quote):** quote unavailable
- **What is different this time:** P2 moved function-level passes earlier to lower memory use; this proposal adds one pass to the O0 pipeline for better code. P2 was withdrawn with no recorded reason.


## 3. Objections a reviewer is likely to raise

### 1. Compile cost on wide deform functions is not measured

fixable · level 2, rubric item 8 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

With jit = on by default, a query on a table with hundreds of columns crosses jit_above_cost but stays below jit_optimize_above_cost. Every execution then pays simplifycfg on a large deform function, the case where compile time matters most. A user notices it only as higher Optimization time in EXPLAIN ANALYZE.

The O0 path serves queries judged too cheap for full optimization, so extra compile time there is felt more than anywhere else. The only numbers are a small single-column query (message #1) and TPC-H Q1 and Q4 (message #4). In message #7, a committer who knows llvmjit said these do not exercise deform routines with many columns. No later message answers this.

Thread: [#1 (Pierre Ducroquet, 2026-01-07)](https://www.postgresql.org/message-id/VS3dpR1Sf5jGnWwoFFJ-_x3GbW7fdmV0arzWPIDfrmbVzewifgu6DsQ7oDa-TAwRz9N2p817j3jGstHwfPOJJxOipbcp-nHdNj3zyxKvC4Q%3D%40pinaraf.info), [#4 ("Matheus Alcantara", 2026-01-22)](https://www.postgresql.org/message-id/DFVG0AV650GW.2CNS5CZ4OG788%40gmail.com), [#7 (Andres Freund, 2026-01-28)](https://www.postgresql.org/message-id/porx6mjfalwefma2f2d76hagxjin3xdgefjsklkzxsyit736ly%4034ubivd7buw2)

**Question:** What do Optimization time and execution time look like for deform functions with about 100 to 1000 columns, with jit_optimize_above_cost = -1?

Counter-check: held. Message #7 asked for deform routines with many columns, and the only later data is a 6-column IR comparison in message #10 with no timings. The O0 path runs for every query above jit_above_cost, so the missing measurement matters.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch changes the LLVM pass strings in llvmjit.c, which must behave well across all supported LLVM versions. |
| performance on the hot path | **yes** | Every query compiled by JIT below jit_optimize_above_cost pays the extra simplifycfg pass at compile time. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What are Optimization time and execution time for deform functions with about 100 to 1000 columns, with jit_optimize_above_cost = -1?
2. Why does simplifycfg remove the stack spills that mem2reg leaves in deform functions?
3. Should mem2reg run before or after simplifycfg, and does the order change compile time or the generated code?
4. Do the gains hold on LLVM versions other than 19 and on architectures other than x86_64?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund, who knows llvmjit, raised the benchmark question in the thread; a committer who knows the LLVM pass pipeline would have to own it.
- **Liability:** The owner signs for JIT compile time on every query below jit_optimize_above_cost, and for the behaviour of the O0 pass string on each supported and future LLVM version.
- **Author follow-up:** The author rebased the patch the same day a reviewer asked (message #3) and moved the entry to a new CommitFest in August 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches only llvmjit.c, adds no user-visible surface, and a reviewer measured TPC-H Q4 about 18% faster. The missing piece is one measurement on wide tables.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` The gain rests on an LLVM side effect that nobody has explained. Refuted (no failure scenario): If a later LLVM release removes the effect, Postgres returns to today's behaviour, which is a lost gain and not a regression. Message #7's maintenance worry concerns replacing default<O3> with an explicit pass list, not adding one long-standing pass to the O0 string.

### Review angles

- prior-attempts: 0 objection(s). The only retrieved precedent [P2] is a different idea in the same area, and it was withdrawn with no recorded reason, so no earlier rejection reason applies. The thread does not point to any earlier discussion of adding simplifycfg at O0.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no syntax, GUC, catalog entry, dump state or psql change, and query results stay the same. Only the internal LLVM pass list below jit_optimize_above_cost changes, and the other angles cover its cost and maintenance.
- replication-ondisk: 0 objection(s). The patch changes only the in-memory LLVM pass list in llvmjit.c. JIT code lives only in the backend that compiles it, so nothing reaches disk, WAL, a standby, the catalog, the wire protocol or a struct that extensions use, and this angle has nothing to object to.
- security: 0 objection(s). The patch only adds the simplifycfg pass to the internal LLVM pipeline used when JIT optimization is below O3. It adds no SQL surface, privilege, setting or new data exposure. Any user who can run a JIT-compiled query already controls when JIT runs, and a pass that removes empty blocks gives no new way to cause denial of service. A security angle has nothing to object to here.
- maintenance-fit: 2 objection(s). The change adds one pass name in llvmjit.c. Its costs fall only on queries that use JIT, and a reviewer has given +1. Neither objection is fatal.
- already-exists: 0 objection(s). Postgres has no cheap way today to get simplifycfg on JIT code without the full default<O3> pipeline. Lowering jit_optimize_above_cost brings O3 and its much higher optimization time, and replacing the jit_provider means a whole new JIT provider. No contrib module or extension offers a configurable pass list, and the patch changes the existing pipeline strings instead of adding a second mechanism.
- demand: 0 objection(s). No demand objection. The patch changes no user-visible surface, and every query compiled by JIT below jit_optimize_above_cost gets the faster code, with a small compile cost that only JIT users pay. Two people measured gains independently: Q4 about 18% faster in message #4, and 7ms in message #1. Whether the benchmarks cover wide tables is a question for the performance angle.

### Precedents retrieved

- P1: [Enable jitlink as an alternative jit linker of legacy Rtdyld and add riscv jitting support](https://commitfest.postgresql.org/patch/3857/) (cf, Returned with feedback, 2023-09-01) - judged not relevant; reason summarised from its thread
- P2: [JIT: Optimize generated functions earlier to lower memory usage](https://commitfest.postgresql.org/patch/2298/) (cf, Withdrawn, 2019-11-12) - used in the review; reason summarised from its thread
- P3: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant; reason summarised from its thread
- P4: [JIT compilation per plan node](https://commitfest.postgresql.org/patch/4837/) (cf, Withdrawn, 2024-08-21) - judged not relevant; reason summarised from its thread
- P5: [JIT compiling expressions/tuple deforming](https://commitfest.postgresql.org/patch/1285/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P6: [jit_warn_above_fraction parameter](https://commitfest.postgresql.org/patch/3573/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P7: [Add llvm version into the version string](https://commitfest.postgresql.org/patch/5202/) (cf, Returned with feedback, 2024-09-24) - judged not relevant; reason summarised from its thread
- P8: [Support multiple password-based authentication tokens concurrently](https://commitfest.postgresql.org/patch/3748/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P9: [Enhancements to passwordcheck](https://commitfest.postgresql.org/patch/1309/) (cf, Returned with feedback, 2017-11-02) - judged not relevant
- P10: [PassDownLimitBound for ForeignScan/CustomScan](https://commitfest.postgresql.org/patch/746/) (cf, Rejected, 2016-09-28) - judged not relevant
- P11: [PassDownLimitBound for ForeignScan/CustomScan](https://commitfest.postgresql.org/patch/1064/) (cf, Returned with feedback, 2017-03-24) - judged not relevant
- P12: [Multiple passwords, interval expirations](https://commitfest.postgresql.org/patch/4432/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P13: [Pass down Limit bounds \[take-2\]](https://commitfest.postgresql.org/patch/866/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P14: [Two pass check for deadlock](https://commitfest.postgresql.org/patch/1602/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P15: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P16: [Revert "TupleHashTable: store additional data along with tuple."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4a07f532b40a64fb4714a3f7ab6063435411edb) (revert, Reverted, 2025-01-13) - judged not relevant
- P17: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P18: [Improve management of SLRU statistics collection.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=81ca8686305c4c62d723ab224ad5c414f350a3a0) (revert, Partially reverted, 2020-05-13) - judged not relevant
- P19: [Allow FunctionScans to pipeline results](https://commitfest.postgresql.org/patch/2372/) (cf, Withdrawn, 2020-03-23) - judged not relevant
- P20: [Batch (pipelined) mode for libpq](https://commitfest.postgresql.org/patch/634/) (cf, Returned with feedback, 2016-11-22) - judged not relevant
- P21: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P22: [Improve overflow checks potentially optimized away by compiler](https://commitfest.postgresql.org/patch/400/) (cf, Returned with feedback, 2015-12-01) - judged not relevant
- P23: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P24: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1024/) (cf, Returned with feedback, 2017-10-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 112 s; cost reported by the CLI: $1.51
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 53 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:50:18.579Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
