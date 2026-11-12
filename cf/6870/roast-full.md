# Design pre-review: LLVM JIT: any JIT-compiled query crashes (SIGILL) on a libLLVM 19 + ASAN build

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6870](https://commitfest.postgresql.org/patch/6870/) (Needs review); authors: Henson Choi (assam258.gmail.com)
- Thread: [LLVM JIT: any JIT-compiled query crashes (SIGILL) on a libLLVM 19 + ASAN build](https://www.postgresql.org/message-id/flat/CAAAe_zD6jGANGZFKnHLKHF8izqmqqJbVe%3DNOuERFwN_Spj5VOA%40mail.gmail.com), started 2026-06-10
- Messages read: 11 of 11
- Patch prose read: v4-0001-Add-a-regression-test-that-runs-a-query-through-t.patch, v4-0002-Raise-an-error-for-a-JIT-type-variable-of-an-unex.patch, v4-0003-Make-the-LLVM-JIT-work-on-an-AddressSanitizer-bui.patch, v4-0004-Exclude-the-variables-in-llvmjit_types.c-from-Add.patch, v1-0001-Add-jit_crash-regression-test-to-force-LLVM-JIT-c.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection stops the proposal at level 1a or 1b: the crash is reproducible, CI demand is shown, and a committer pushed 0004 as 0607aa61090. The one surviving objection, maintenance-fit-2, is a fixable level 2 point about whether a source pragma or filtering instrumentation flags from the bitcode flags is the better long-term fix. What remains (0002, the jit test, the build-system approach) is for a committer who knows the JIT build to decide.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review cannot judge whether the 0002 check rejects any valid non-sanitizer type, or whether the jit test output is stable on builds without LLVM. Only 3 of 11 messages were read in full.

## The proposal, restated

**Problem.** On a build with -fsanitize=address, every JIT-compiled query crashes or hangs the backend, even SELECT 1. Linux backends die of SIGILL or SIGSEGV inside LLVM debug-info code, and macOS loops forever inside LLVM. The regression tests miss it because jit is off by default, and the crash breaks the CI AddressSanitizer job for other patches.

**Proposed design.** AddressSanitizer wraps each global variable in llvmjit_types.c in an unnamed struct { T, [N x i8] }, so llvm_pg_var_type() hands the JIT a wrong type and the JIT builds invalid IR. v4 makes llvm_pg_var_type() raise an error for an unnamed struct type, and either takes T out of the wrapper (0003) or excludes the file's variables from instrumentation with a clang pragma (0004). It also adds a jit regression test.

**User-visible surface.**
- New regression test jit (sql/jit.sql, expected/jit.out) in parallel_schedule; it sets jit = on and jit_above_cost = 0.
- New error: "variable TypeSizeT in llvmjit_types.c is not of its declared type" for a JIT type variable of an unexpected form.
- Earlier versions (v2, v3) added a meson warning() and a configure warning when sanitizer flags are filtered from bitcode flags; v4 drops these.

**What the author claims.**
- The crash reproduces on plain master with a trivial query and is not specific to the Row Pattern Recognition patch.
- It reproduces on LLVM 19, 20 and 21, on x86_64 and arm64, on Linux and macOS.
- With 0002 the crash becomes an ERROR; with 0003 or 0004 the jit test passes, and Greg Burd confirmed this on Debian 13 and aarch64.
- 0003 depends on the internal shape of the AddressSanitizer wrapper; 0004 does not but adds a clang-specific pragma.
- Committing the test alone breaks the CI AddressSanitizer build.

**Thread state.** Andres Freund pushed 0004 to 17 through master as 0607aa61090, and Greg Burd and Matheus Alcantara reviewed earlier versions. Open items: 0002, 0003, the test, backpatching to older branches, and a possible build-system approach (filtering c_args or replacing 1039352e1); the CommitFest entry still shows Needs review and needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-10; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Per-sanitizer source pragmas may not scale to other instrumentation

fixable · level 2, rubric item 5 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A developer builds with another sanitizer that also changes global types in llvmjit_types.c, such as hwaddress. The 0004 pragma does not cover that sanitizer, so JIT queries fail again, and the developer finds out only when a JIT test runs. Each new sanitizer would then need its own pragma or unwrapping rule.

The root cause is that 1039352e1 passes all c_args, including instrumentation flags, into bitcode compilation (message #10). 0002 and 0004 address one effect of one sanitizer at the point where types are read. Message #10 names filtering c_args, or replacing 1039352e1, as possible alternatives, and v2/v3 (messages #5, #7) already did this filtering. The material does not show whether UBSan or other sanitizers cause similar breakage.

Thread: [#5 (Henson Choi, 2026-06-12)](https://www.postgresql.org/message-id/CAAAe_zDtwoL8KaC_cpK4rU9jCrLxtGJ%3DTwLogdfH1PeE3_GT9Q%40mail.gmail.com), [#7 (Henson Choi, 2026-06-15)](https://www.postgresql.org/message-id/CAAAe_zBX5uV9K0ikuROLgdNvDCgGqHRskT-73L%2BoX9%3D3aXR2AQ%40mail.gmail.com), [#10 (Andres Freund, 2026-11-07)](https://www.postgresql.org/message-id/jlc4cv33seses5ttgnlzlvidvbzwsexwhxzc67db3frftf76m5%40ttcoyo2vttkw)

**Question:** Does any sanitizer besides AddressSanitizer change the types in llvmjit_types.c bitcode, and if so, why is a source pragma better than filtering instrumentation flags out of bitcode_cflags?

Counter-check: held. Message #10 leaves open the choice between the pragma and filtering c_args, and the thread does not say whether other sanitizers change these types. The hwaddress case is unverified, so this stands as a question for the next version.

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
| backpatchability | **yes** | The entry targets stable branches; 0004 went to 17 through master, and backpatching of the remaining pieces is open. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A clang-specific pragma in llvmjit_types.c, a new check in llvm_pg_var_type(), and a new jit regression test need upkeep. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. After 0607aa61090, what will the next version contain: 0002, the jit test, both, or a build-system change instead?
2. Does any sanitizer besides AddressSanitizer, such as UBSan, change the variable types in the llvmjit_types.c bitcode?
3. Why is the source pragma preferable to filtering instrumentation flags out of the bitcode flags, as v2 and v3 did?
4. How long does the jit test run, and what does it do on a build without LLVM?
5. Is the -g change for autoconf BITCODE_CFLAGS from v2 and v3 still proposed?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund pushed 0004 as 0607aa61090; the rest needs someone who knows the LLVM JIT provider and the bitcode build flags in meson and autoconf.
- **Liability:** The owner signs for a jit test that must stay stable across LLVM versions and build configurations on every backpatched branch, and for an error check tied to how compilers shape the type variables.
- **Author follow-up:** The author posted four versions and moved the entry through three CommitFests; the entry needs a rebase since 2026-11-09 and no later version is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patches are small and touch only the JIT provider and the regression schedule, and the main fix is already committed. The benefit goes to developers and CI runs that use sanitizer builds, and the entry needs a rebased version that states what is still proposed.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` 0003 would add unreachable code tied to AddressSanitizer internals. Refuted (already handled): Message #8 offers 0003 and 0004 as alternatives and says only one is needed. Andres Freund picked 0004 in message #10, and nothing in the thread proposes keeping 0003 as well.
- `already-exists-1` 0003 duplicates the fix already committed as 0004. Refuted (already handled): Same point as maintenance-fit-1. Message #8 presents 0003 only as an alternative to 0004, and the commit of 0004 in message #10 settles that choice without anyone asking for both.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P6] and [P1] are in neighbouring areas but are different ideas, and their recorded reasons do not apply here. [P6] objected to zeroing all palloc'd memory at runtime, and this proposal adds no runtime cost to normal builds. [P1] asked for target-specific settings to live in LLVM, but the AddressSanitizer wrapper is valid compiler output and the faulty assumption sits in llvm_pg_var_type(). The thread cites only one earlier change, 1039352e1 in message #10, which added c_args to the bitcode flags; nobody describes an earlier proposal of this fix.
- standards-compat: 0 objection(s). The proposal adds no syntax, GUC, catalog object, psql feature or dump state, and no existing user behaviour changes; 0004 is already committed. What remains is a regression test and an internal error for a malformed JIT type, neither of which touches the user-facing contract this angle covers.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state. It changes how llvmjit.c reads LLVM types from in-memory bitcode and adds a regression test, with no change to on-disk format, WAL, catalog, wire protocol or the extension ABI. JIT code is built per backend and never written to disk or replicated, so pg_upgrade, standbys and subscribers are not affected.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It fixes how the JIT reads type information on AddressSanitizer test builds, adds an error check, and adds a regression test. Setting jit = on for a session is already allowed for any user, and nothing in it changes what any role can do or see.
- maintenance-fit: 2 objection(s). The jit test and the 0002 check look cheap. They cost nothing for users who do not build with sanitizers or enable jit. A committer who knows the area (Andres Freund) is engaged and has already pushed 0004.
- already-exists: 1 objection(s). 0002 and the jit test do not duplicate an existing feature closely enough to object; Andres notes in message #10 that no test forces JIT on, and he asks for a proper test.
- demand: 0 objection(s). The thread shows real demand: CI's AddressSanitizer job crashed on several patches (Row Pattern Recognition in message #11, a pageinspect JIT test in message #9), and three independent people hit it. A committer pushed 0004, and the remaining test and 0002 cost little and help everyone who runs CI.

### Precedents retrieved

- P1: [Enable jitlink as an alternative jit linker of legacy Rtdyld and add riscv jitting support](https://commitfest.postgresql.org/patch/3857/) (cf, Returned with feedback, 2023-09-01) - used in the review; reason summarised from its thread
- P2: [Add llvm version into the version string](https://commitfest.postgresql.org/patch/5202/) (cf, Returned with feedback, 2024-09-24) - judged not relevant; reason summarised from its thread
- P3: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P4: [Propagate sanity checks of ProcessUtility() to standard_ProcessUtility()?](https://commitfest.postgresql.org/patch/4877/) (cf, Withdrawn, 2024-05-18) - judged not relevant; reason summarised from its thread
- P5: [psql \setfileref - new psql variable type - reference to file](https://commitfest.postgresql.org/patch/787/) (cf, Returned with feedback, 2016-12-15) - judged not relevant; reason summarised from its thread
- P6: [Make PostgreSQL sanitizers-friendly (and prevent information disclosure)](https://commitfest.postgresql.org/patch/655/) (cf, Returned with feedback, 2016-09-02) - used in the review; reason summarised from its thread
- P7: [Revert last change (CFLAGS+=-g). Probably was a mistake...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=154ccb60408a0e389ddcc79f54fc1e1bc58235c0) (revert, Reverted, 2002-01-08) - judged not relevant
- P8: [Increasing regression test coverage based on GCOV analysis](https://commitfest.postgresql.org/patch/2257/) (cf, Withdrawn, 2019-08-24) - judged not relevant; reason summarised from its thread
- P9: [audo-detect and use -moutline-atomics compilation flag for aarch64](https://commitfest.postgresql.org/patch/2637/) (cf, Rejected, 2020-09-29) - judged not relevant; reason summarised from its thread
- P10: [Speed up clean meson builds by ~25%](https://commitfest.postgresql.org/patch/4914/) (cf, Rejected, 2024-05-20) - judged not relevant
- P11: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P12: [Revert "Test replay of regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dccee0f2b72648b8b3c42883a7d99210eb813f30) (revert, Reverted, 2022-01-14) - judged not relevant
- P13: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - judged not relevant
- P14: [xml2: add test for coverage](https://commitfest.postgresql.org/patch/3833/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P15: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P16: [Revert "Remove reset of testtablespace from pg_regress on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61be85afabba0cd85bb1bcaacbea8efa8641f564) (revert, Reverted, 2020-07-10) - judged not relevant
- P17: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant
- P18: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - judged not relevant
- P19: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant
- P20: [Direct converting numeric types to bool](https://commitfest.postgresql.org/patch/1577/) (cf, Rejected, 2018-03-01) - judged not relevant
- P21: [Revert temporary patch that made synchronous_commit default to OFF.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f243a1a1bd836ed45f6e37a3843868bd9bbecad7) (revert, Reverted, 2007-09-10) - judged not relevant
- P22: [Mark function arguments of type "T *" as "const T *" where possible](https://commitfest.postgresql.org/patch/6302/) (cf, Withdrawn, 2026-01-13) - judged not relevant
- P23: [old_snapshot: add test for coverage](https://commitfest.postgresql.org/patch/3805/) (cf, Returned with feedback, 2022-11-25) - judged not relevant
- P24: [Global temporary table](https://commitfest.postgresql.org/patch/2349/) (cf, Returned with feedback, 2022-06-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 129 s; cost reported by the CLI: $1.62
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 61 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:06:17.810Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
