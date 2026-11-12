# Design pre-review: JIT inlining SIGSEGV in MemoryBuffer::getFile with LLVM 17 / GCC 13

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7289](https://commitfest.postgresql.org/patch/7289/); [thread](https://www.postgresql.org/message-id/flat/dc9fb68c682c4eec9d18f1007bd568f8%40localhost.localdomain), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). No objection to the idea or the design survived. The patch moves one call onto the C API that load_module() already uses, in one file. The only surviving objection, maintenance-fit-2, is fixable: message #1 states the std::optional<Align> layout as the cause but does not confirm it. That gap affects the commit message and the back-patch decision, so a committer who knows the JIT code has to judge it.

Not checked: Without the code, the error path for a missing or unreadable summary file and the ownership of the C API memory buffer cannot be checked. The stated cause of the crash is also unverified.

## Prior attempts

None found before 2026-08-19.

## Objections a reviewer is likely to raise

### 1. Root cause unconfirmed, so the change may hide a packaging fault

The real cause is something else, such as a libstdc++ or header mismatch in the openSUSE packages. The patch hides it at this site. The fault then surfaces at another C++ call, and maintainers debug llvmjit code that has already been patched on a theory nobody confirmed.

Message #1 blames the std::optional<Align> layout but gives no disassembly and no build with matching compilers. It also does not say which compiler built libLLVM.so.17. The standalone C++ program crashing on /etc/hostname points toward a toolchain or packaging defect outside Postgres. That kind of defect may belong with the distro.

**Question:** Does the crash go away when llvmjit.so is built with the compiler and C++ library that built the openSUSE libLLVM.so.17?

<sub>fixable, level 1a, problem statement; messages [#1](https://www.postgresql.org/message-id/dc9fb68c682c4eec9d18f1007bd568f8%40localhost.localdomain); `maintenance-fit-2`</sub>

## Questions for the author

1. Does the crash go away when llvmjit.so is built with the compiler and C++ library that built the openSUSE libLLVM.so.17?
2. Which compiler and C++ standard library built the openSUSE libLLVM.so.17 package?
3. Which branches should get the fix, and on which LLVM versions other than 17 and 18 was the patch tested?
4. Does a missing or unreadable postgres.index.bc produce the same behaviour with the C API as with MemoryBuffer::getFile()?
5. Were the other C++ LLVM calls in llvmjit checked for by-value or defaulted standard-library arguments?

## Touch points

- **backpatchability:** The entry is a crash fix for LLVM 17 builds, and message #1 does not say which branches should get it.
- **maintenance burden ("who maintains this in 10 years"):** The change is in llvmjit_inline.cpp, which depends on LLVM interfaces the project does not control.

## Who would commit it, and when

- Owner: silent. A committer who knows the LLVM JIT provider and its inlining code. Message #1 copies two people for this area, but nobody has replied.
- Priority: needs champion; audience niche; review cost low. The crash affects builds where Postgres and the distro libLLVM use different compilers and JIT inlining runs. The patch is 2.7 KB in one file and copies load_module(), but no one has replied since 2026-08-19.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
