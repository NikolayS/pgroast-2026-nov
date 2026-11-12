# Design pre-review: JIT inlining SIGSEGV in MemoryBuffer::getFile with LLVM 17 / GCC 13

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7289](https://commitfest.postgresql.org/patch/7289/) (Needs review); authors: Yuriy Grigoryev (ju.grigorev)
- Thread: [\[PATCH\] JIT inlining SIGSEGV in MemoryBuffer::getFile with LLVM 17 / GCC 13](https://www.postgresql.org/message-id/flat/dc9fb68c682c4eec9d18f1007bd568f8%40localhost.localdomain), started 2026-08-19
- Messages read: 1 of 1
- Patch prose read: v1-0001-jit-summary-c-api.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

No objection to the idea or the design survived. The patch moves one call onto the C API that load_module() already uses, in one file. The only surviving objection, maintenance-fit-2, is fixable: message #1 states the std::optional<Align> layout as the cause but does not confirm it. That gap affects the commit message and the back-patch decision, so a committer who knows the JIT code has to judge it.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the error path for a missing or unreadable summary file and the ownership of the C API memory buffer cannot be checked. The stated cause of the crash is also unverified.

## The proposal, restated

**Problem.** With LLVM 17 and GCC 13 on openSUSE Leap 15.6, JIT inlining crashes the backend with signal 11 inside llvm::MemoryBuffer::getFile(), called from llvm_load_summary(). With jit_above_cost and jit_inline_above_cost set to 0, about a hundred check-world tests fail. It affects builds where Postgres and the distro libLLVM use different compilers.

**Proposed design.** llvm_load_summary() reads postgres.index.bc with LLVMCreateMemoryBufferWithContentsOfFile() instead of the C++ MemoryBuffer::getFile(), as load_module() already does. The author blames the std::optional<Align> default argument, built at the call site, whose layout a GCC 13 build and a distro libLLVM.so.17 need not share. The change touches only src/backend/jit/llvm/llvmjit_inline.cpp.

**What the author claims.**
- Skipping llvm_inline() makes check-world pass, and Ubuntu with LLVM 18 does not crash.
- A small C++ program calling getFile("/etc/hostname") crashes the same way, while the C API works on the 2.4MB index.
- With the patch, the boolean join returns 4 rows and 6 functions are inlined.
- The crash is the other face of the ABI problem in BUG #18254, where FreeBSD with LLVM 16 could not resolve the getFile symbol.
- The C API is a stable contract between compilers, so the fix belongs in tree and is not a local toolchain problem.

**Thread state.** Only the first message is visible. The author copies Andres and Thomas, but nobody has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Root cause unconfirmed, so the change may hide a packaging fault

fixable · level 1a, rubric item 1 · problem statement, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

The real cause is something else, such as a libstdc++ or header mismatch in the openSUSE packages. The patch hides it at this site. The fault then surfaces at another C++ call, and maintainers debug llvmjit code that has already been patched on a theory nobody confirmed.

Message #1 blames the std::optional<Align> layout but gives no disassembly and no build with matching compilers. It also does not say which compiler built libLLVM.so.17. The standalone C++ program crashing on /etc/hostname points toward a toolchain or packaging defect outside Postgres. That kind of defect may belong with the distro.

Thread: [#1 (Grigorev Jurij, 2026-08-19)](https://www.postgresql.org/message-id/dc9fb68c682c4eec9d18f1007bd568f8%40localhost.localdomain)

**Question:** Does the crash go away when llvmjit.so is built with the compiler and C++ library that built the openSUSE libLLVM.so.17?

Counter-check: held. The commit message states the std::optional<Align> layout as the cause, but message #1 shows no disassembly, no matching-compiler build, and not which compiler built libLLVM.so.17. The change is safe either way, so the gap affects the recorded rationale and the back-patch decision.

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
| backpatchability | **yes** | The entry is a crash fix for LLVM 17 builds, and message #1 does not say which branches should get it. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The change is in llvmjit_inline.cpp, which depends on LLVM interfaces the project does not control. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the crash go away when llvmjit.so is built with the compiler and C++ library that built the openSUSE libLLVM.so.17?
2. Which compiler and C++ standard library built the openSUSE libLLVM.so.17 package?
3. Which branches should get the fix, and on which LLVM versions other than 17 and 18 was the patch tested?
4. Does a missing or unreadable postgres.index.bc produce the same behaviour with the C API as with MemoryBuffer::getFile()?
5. Were the other C++ LLVM calls in llvmjit checked for by-value or defaulted standard-library arguments?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the LLVM JIT provider and its inlining code. Message #1 copies two people for this area, but nobody has replied.
- **Liability:** The owner signs for a back-patched change in the JIT inlining path and for a commit message that records an unconfirmed cause. The code change is one call in one file.
- **Author follow-up:** The author asks for feedback in message #1 and created the CommitFest entry about three weeks later. No later sign.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The crash affects builds where Postgres and the distro libLLVM use different compilers and JIT inlining runs. The patch is 2.7 KB in one file and copies load_module(), but no one has replied since 2026-08-19.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` One call site fixed, the rest of llvmjit keeps the same ABI exposure. Refuted (no failure scenario): Message #1 reports that with the patch getModuleSummaryIndex() works and 6 functions are inlined, so summary parsing and IRMover ran on this toolchain. The patch copies what load_module() already does and adds no support promise; a further crash is a guess.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P2] covers JITLink and riscv target settings and added a C++ wrapper where LLVM has no C API; this patch moves one call onto an existing C API, so the reason recorded for [P2] does not apply. Message #1 points to BUG #18254, a link failure on the same getFile symbol, but describes it as a bug report, not a rejected fix.
- standards-compat: 0 objection(s). The patch adds no SQL syntax, GUC, catalog object, psql behaviour or dump content, and it touches one internal file only. Query results and settings stay the same for existing users, so this angle finds nothing to object to. Whether a missing or unreadable summary file is handled the same way with the C API cannot be checked without the code.
- replication-ondisk: 0 objection(s). The patch changes only how llvmjit_inline.cpp reads postgres.index.bc into memory. It adds no on-disk format, WAL record, catalog change, protocol message or Postgres struct or hook. Nothing persistent or cross-node is involved, and the change does not block backpatching for ABI reasons.
- security: 0 objection(s). The patch changes how llvm_load_summary() reads postgres.index.bc, a file installed by Postgres itself, and adds no user-visible surface, privilege or setting. Nothing in it changes who can trigger JIT or what a non-superuser can see, so this angle has no objection; the error path on an unreadable summary file can only be checked in the code.
- maintenance-fit: 2 objection(s). The change itself copies the existing practice in load_module() and touches one file, which lowers confidence in rejection. Both objections concern the rationale and its scope, not the change itself.
- already-exists: 0 objection(s). Building Postgres with the compiler that built libLLVM, or setting jit_inline_above_cost = -1, avoids the crash, but the second gives up inlining and the first is not how distributions package Postgres. The patch reuses the C API call that load_module() already uses, so it adds no new mechanism, and P2's reason (the work belongs in LLVM) does not apply to a caller-side ABI mismatch.
- demand: 0 objection(s). The demand angle finds no objection. The proposal fixes a backend crash: about a hundred check-world failures on default openSUSE Leap 15.6 devel packages, with an outside report of the same getFile() crash (message #1). The change touches one file and follows the C API call that load_module() already uses, so non-users pay almost nothing. The design and root-cause questions belong to other angles.

### Precedents retrieved

- P1: [Add llvm version into the version string](https://commitfest.postgresql.org/patch/5202/) (cf, Returned with feedback, 2024-09-24) - judged not relevant; reason summarised from its thread
- P2: [Enable jitlink as an alternative jit linker of legacy Rtdyld and add riscv jitting support](https://commitfest.postgresql.org/patch/3857/) (cf, Returned with feedback, 2023-09-01) - used in the review; reason summarised from its thread
- P3: [JIT compiling expressions/tuple deforming](https://commitfest.postgresql.org/patch/1285/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P4: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant; reason summarised from its thread
- P5: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P6: [pg_buffercache: Add per-relation summary stats](https://commitfest.postgresql.org/patch/6555/) (cf, Returned with feedback, 2026-05-04) - judged not relevant; reason summarised from its thread
- P7: [jit_warn_above_fraction parameter](https://commitfest.postgresql.org/patch/3573/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P8: [JIT compilation per plan node](https://commitfest.postgresql.org/patch/4837/) (cf, Withdrawn, 2024-08-21) - judged not relevant; reason summarised from its thread
- P9: [Revert change to ALTER TABLESPACE summary.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=155c0f24b7d370eb57fb8dde0447e11e41c405c2) (revert, Reverted, 2014-11-14) - judged not relevant
- P10: [SQL Inlining Behavior](https://commitfest.postgresql.org/patch/1965/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P11: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P12: [abidiff tests](https://commitfest.postgresql.org/patch/4358/) (cf, Withdrawn, 2024-03-14) - judged not relevant
- P13: [Fix runtime errors from -fsanitize=undefined](https://commitfest.postgresql.org/patch/2144/) (cf, Rejected, 2019-08-13) - judged not relevant
- P14: [BUG Segfault triggered by reentrancy in ri_triggers.c](https://commitfest.postgresql.org/patch/6805/) (cf, Withdrawn, 2026-05-29) - judged not relevant
- P15: [Cleanup gcc trick with varattrib_1b_e in VARATT_EXTERNAL_GET_POINTER()](https://commitfest.postgresql.org/patch/5807/) (cf, Withdrawn, 2025-07-14) - judged not relevant
- P16: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P17: [Fix possible segfault in walreceiver](https://commitfest.postgresql.org/patch/2873/) (cf, Withdrawn, 2020-12-17) - judged not relevant
- P18: [Adding Support for Copy callback functionality on COPY TO api](https://commitfest.postgresql.org/patch/2660/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P19: [BUG Segfault triggered by reentrancy in ri_triggers.c](https://commitfest.postgresql.org/patch/6815/) (cf, Withdrawn, 2026-06-01) - judged not relevant
- P20: [Revert workarounds for -Wmissing-braces false positives on old GCC](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=618c64ffd3967cb5313b4b11e1e1043a074f2139) (revert, Reverted, 2025-03-20) - judged not relevant
- P21: [Introduce the ability to enable/disable indexes](https://commitfest.postgresql.org/patch/5267/) (cf, Withdrawn, 2024-09-26) - judged not relevant
- P22: [Introduce the ability to enable/disable indexes using ALTER INDEX](https://commitfest.postgresql.org/patch/5274/) (cf, Withdrawn, 2025-09-08) - judged not relevant
- P23: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P24: [Clarify ItemPointers for Table AM API](https://commitfest.postgresql.org/patch/3108/) (cf, Withdrawn, 2021-06-18) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1a objection (maintenance-fit-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 86 s; cost reported by the CLI: $0.72
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:37:40.768Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
