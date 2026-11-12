# Design pre-review: LLVM JIT: any JIT-compiled query crashes (SIGILL) on a libLLVM 19 + ASAN build

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6870](https://commitfest.postgresql.org/patch/6870/); [thread](https://www.postgresql.org/message-id/flat/CAAAe_zD6jGANGZFKnHLKHF8izqmqqJbVe%3DNOuERFwN_Spj5VOA%40mail.gmail.com), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection stops the proposal at level 1a or 1b: the crash is reproducible, CI demand is shown, and a committer pushed 0004 as 0607aa61090. The one surviving objection, maintenance-fit-2, is a fixable level 2 point about whether a source pragma or filtering instrumentation flags from the bitcode flags is the better long-term fix. What remains (0002, the jit test, the build-system approach) is for a committer who knows the JIT build to decide.

Not checked: Without the code, the review cannot judge whether the 0002 check rejects any valid non-sanitizer type, or whether the jit test output is stable on builds without LLVM. Only 3 of 11 messages were read in full.

## Prior attempts

None found before 2026-06-10.

## Objections a reviewer is likely to raise

### 1. Per-sanitizer source pragmas may not scale to other instrumentation

A developer builds with another sanitizer that also changes global types in llvmjit_types.c, such as hwaddress. The 0004 pragma does not cover that sanitizer, so JIT queries fail again, and the developer finds out only when a JIT test runs. Each new sanitizer would then need its own pragma or unwrapping rule.

The root cause is that 1039352e1 passes all c_args, including instrumentation flags, into bitcode compilation (message #10). 0002 and 0004 address one effect of one sanitizer at the point where types are read. Message #10 names filtering c_args, or replacing 1039352e1, as possible alternatives, and v2/v3 (messages #5, #7) already did this filtering. The material does not show whether UBSan or other sanitizers cause similar breakage.

**Question:** Does any sanitizer besides AddressSanitizer change the types in llvmjit_types.c bitcode, and if so, why is a source pragma better than filtering instrumentation flags out of bitcode_cflags?

<sub>fixable, level 2, design; messages [#5](https://www.postgresql.org/message-id/CAAAe_zDtwoL8KaC_cpK4rU9jCrLxtGJ%3DTwLogdfH1PeE3_GT9Q%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAAAe_zBX5uV9K0ikuROLgdNvDCgGqHRskT-73L%2BoX9%3D3aXR2AQ%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/jlc4cv33seses5ttgnlzlvidvbzwsexwhxzc67db3frftf76m5%40ttcoyo2vttkw); `maintenance-fit-2`</sub>

## Questions for the author

1. After 0607aa61090, what will the next version contain: 0002, the jit test, both, or a build-system change instead?
2. Does any sanitizer besides AddressSanitizer, such as UBSan, change the variable types in the llvmjit_types.c bitcode?
3. Why is the source pragma preferable to filtering instrumentation flags out of the bitcode flags, as v2 and v3 did?
4. How long does the jit test run, and what does it do on a build without LLVM?
5. Is the -g change for autoconf BITCODE_CFLAGS from v2 and v3 still proposed?

## Touch points

- **backpatchability:** The entry targets stable branches; 0004 went to 17 through master, and backpatching of the remaining pieces is open.
- **maintenance burden ("who maintains this in 10 years"):** A clang-specific pragma in llvmjit_types.c, a new check in llvm_pg_var_type(), and a new jit regression test need upkeep.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund pushed 0004 as 0607aa61090; the rest needs someone who knows the LLVM JIT provider and the bitcode build flags in meson and autoconf.
- Priority: proceed; audience niche; review cost low. The patches are small and touch only the JIT provider and the regression schedule, and the main fix is already committed. The benefit goes to developers and CI runs that use sanitizer builds, and the entry needs a rebased version that states what is still proposed.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
