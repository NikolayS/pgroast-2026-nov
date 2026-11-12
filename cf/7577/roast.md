# Design pre-review: Stale hash entries when reusing an auxiliary resource owner

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7577](https://commitfest.postgresql.org/patch/7577/); [thread](https://www.postgresql.org/message-id/flat/CAPsk3_BD6Sy3_KHo5Mka1yxLBL4m%3Dd3YHbw-%3DA6-oLAEmiHZXQ%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real and nobody disputes it, and a committer who knows the area asked for the v4 design. Two fixable design points survive: standards-compat-1 (which part of v4 goes to stable branches) and standards-compat-2 (what ResourceOwnerReset() checks when resources are still held). Neither stops the idea, so no level fails.

Not checked: Without the code, I could not check what ResourceOwnerReset() asserts, or whether clearing slots before the release callbacks is safe when a callback errors. The benchmarks cover one resource kind and one pgbench workload.

## Prior attempts

None found before 2026-11-02.

## Objections a reviewer is likely to raise

### 1. Stable-branch target, but v4 adds a public API to resowner.h

If v4 is back-patched, an extension built against a new minor release can call ResourceOwnerReset(). That extension then fails to load with an undefined symbol on a server still running an older minor of the same major version. Packagers and users find out at load time, after the upgrade order goes wrong.

The CommitFest entry targets "stable", and v1 said "Backpatch-through: 17". v4 adds a new exported function and changes the semantics of the generic release path, as Andres asked in message #4. No message says which part is meant for back branches. A committer would likely want a minimal fix for 17 and later, such as clearing slots in ResourceOwnerSort() and bulk release, and the new API on master only.

**Question:** Which part of v4 goes to stable branches, and can the back-branch fix work without adding ResourceOwnerReset() to resowner.h?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAPsk3_BD6Sy3_KHo5Mka1yxLBL4m%3Dd3YHbw-%3DA6-oLAEmiHZXQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/duhzrnzhykvs2dzmzejrlqtceto7ptj5r7rtm2ues4vuctsnzm%40h7dx6bovzax4), [#5](https://www.postgresql.org/message-id/CAPsk3_As%3D%2BDSX3t_WfzWJXHESyz%2B0%3DF%3DbbZ9Q-i4VTykq5_96A%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAPsk3_DL5GsjyWKis6O3obxRk58o1LqvPfmT8qyPi-37%3D1rkQQ%40mail.gmail.com); `standards-compat-1`</sub>

### 2. "Reset" borrows the MemoryContext verb but not its contract

An extension author knows that MemoryContextReset() frees what the context holds, and calls ResourceOwnerReset() on an owner that still holds buffer pins or locks. In a build without assertions, the owner is reused and those resources are no longer tracked. The leak shows up later as a pin or lock that is never released.

Message #5 names the function by analogy with MemoryContext. But the README hunk says all resources must be released first, so the verb means something else here. The material does not say whether the function errors, asserts, or does nothing when the owner still holds resources, or when release has not finished. For a public API, a committer would likely ask for a name or a check that makes misuse fail loudly in production builds.

**Question:** What does ResourceOwnerReset() do in a build without assertions when the owner or a child still holds resources or has not finished all three release phases?

<sub>fixable, level 2, ABI / extension API; messages [#4](https://www.postgresql.org/message-id/duhzrnzhykvs2dzmzejrlqtceto7ptj5r7rtm2ues4vuctsnzm%40h7dx6bovzax4), [#5](https://www.postgresql.org/message-id/CAPsk3_As%3D%2BDSX3t_WfzWJXHESyz%2B0%3DF%3DbbZ9Q-i4VTykq5_96A%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Which part of v4 goes to stable branches, and can the back-branch fix work without adding ResourceOwnerReset() to resowner.h?
2. What does ResourceOwnerReset() do in a build without assertions when the owner or a child still holds resources or has not finished all three release phases?
3. How does clearing during sort and bulk release perform for transactions that hold many hashed resources of several kinds?
4. How does ResourceOwnerReset() fit with Rahila Syed's proposal to release all three phases in one call?

## Touch points

- **ABI / extension API:** v4 adds the exported function ResourceOwnerReset() to src/include/utils/resowner.h.
- **backpatchability:** The CommitFest entry targets stable and v1 said "Backpatch-through: 17", but v4 adds a public function.
- **maintenance burden ("who maintains this in 10 years"):** A public reset operation in resowner.c must stay correct as the owner's state changes.
- **performance on the hot path:** ResourceOwnerSort() and bulk release now clear hash slots for every owner, not only auxiliary ones.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund asked in message #4 for the generic helper and for clearing in ResourceOwnerSort(). The owner must know resowner.c and the auxiliary-process cleanup path.
- Priority: proceed; audience niche; review cost low. The patch is about 12 KiB and local to resowner.c and its test module. It fixes a double release or hang in reused auxiliary owners, and the remaining points are a back-branch plan and the reset precondition check.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
