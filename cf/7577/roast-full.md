# Design pre-review: Stale hash entries when reusing an auxiliary resource owner

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7577](https://commitfest.postgresql.org/patch/7577/) (Needs review); authors: Jingtang Zhang (mrdrivingduck)
- Thread: [Stale hash entries when reusing an auxiliary resource owner](https://www.postgresql.org/message-id/flat/CAPsk3_BD6Sy3_KHo5Mka1yxLBL4m%3Dd3YHbw-%3DA6-oLAEmiHZXQ%40mail.gmail.com), started 2026-11-02
- Messages read: 7 of 7
- Patch prose read: v4-aux-resource-owner-reset.patch, v1-aux-resource-owner-reuse.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real and nobody disputes it, and a committer who knows the area asked for the v4 design. Two fixable design points survive: standards-compat-1 (which part of v4 goes to stable branches) and standards-compat-2 (what ResourceOwnerReset() checks when resources are still held). Neither stops the idea, so no level fails.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check what ResourceOwnerReset() asserts, or whether clearing slots before the release callbacks is safe when a callback errors. The benchmarks cover one resource kind and one pgbench workload.

## The proposal, restated

**Problem.** Sorting and bulk release leave stale entries in a resource owner's hash table, but ReleaseAuxProcessResources() makes the auxiliary owner reusable without clearing them. On reuse, old entries can be released a second time, or insertion can loop forever because no slot is free. The author traces the bug to b8bff07daa8.

**Proposed design.** v4 adds a public ResourceOwnerReset() that restores an already released owner and its descendants for reuse and keeps their storage. Auxiliary-process cleanup now uses it. ResourceOwnerSort() clears moved source slots during compaction, and bulk release clears each retired hash entry before its release callback, so reset needs no full-table clearing pass.

**User-visible surface.**
- New C API function ResourceOwnerReset(), callable after all three release phases, which resets an owner and its child owners for reuse.
- src/backend/utils/resowner/README documents the new reset operation.
- test_resowner test module gains new SQL-callable test functions.

**What the author claims.**
- Without the fix, a reused auxiliary owner can release old resources twice or hang in ResourceOwnerAddToHash().
- The v2 and later regression tests fail promptly with duplicate-release errors when only the fix is reverted, instead of hanging.
- In microbenchmarks, clearing during sort and release costs little, except at 16,384 resources (about 240 vs 256 microseconds).
- Clearing the full table in reset (option B) grows with the earlier peak, up to about 67 microseconds at a 65,536-entry peak.
- pgbench -S showed no consistent throughput change (-1.60% and +0.54% in two runs).

**Thread state.** Andres Freund asked for a generic reset helper and for clearing in ResourceOwnerSort(), and Rahila Syed asked for benchmarks and plans a detailed review. The author answered with benchmarks and v4, and nobody has replied to v4 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-02; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Stable-branch target, but v4 adds a public API to resowner.h

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `standards-compat-1`

If v4 is back-patched, an extension built against a new minor release can call ResourceOwnerReset(). That extension then fails to load with an undefined symbol on a server still running an older minor of the same major version. Packagers and users find out at load time, after the upgrade order goes wrong.

The CommitFest entry targets "stable", and v1 said "Backpatch-through: 17". v4 adds a new exported function and changes the semantics of the generic release path, as Andres asked in message #4. No message says which part is meant for back branches. A committer would likely want a minimal fix for 17 and later, such as clearing slots in ResourceOwnerSort() and bulk release, and the new API on master only.

Thread: [#1 (Jingtang Zhang, 2026-11-02)](https://www.postgresql.org/message-id/CAPsk3_BD6Sy3_KHo5Mka1yxLBL4m%3Dd3YHbw-%3DA6-oLAEmiHZXQ%40mail.gmail.com), [#4 (Andres Freund, 2026-11-02)](https://www.postgresql.org/message-id/duhzrnzhykvs2dzmzejrlqtceto7ptj5r7rtm2ues4vuctsnzm%40h7dx6bovzax4), [#5 (Jingtang Zhang, 2026-11-03)](https://www.postgresql.org/message-id/CAPsk3_As%3D%2BDSX3t_WfzWJXHESyz%2B0%3DF%3DbbZ9Q-i4VTykq5_96A%40mail.gmail.com), [#7 (Jingtang Zhang, 2026-11-08)](https://www.postgresql.org/message-id/CAPsk3_DL5GsjyWKis6O3obxRk58o1LqvPfmT8qyPi-37%3D1rkQQ%40mail.gmail.com)

**Question:** Which part of v4 goes to stable branches, and can the back-branch fix work without adding ResourceOwnerReset() to resowner.h?

Counter-check: held. The CommitFest entry targets stable and v1 said "Backpatch-through: 17", but v4 adds an exported function. No message says which part goes to back branches.

### 2. "Reset" borrows the MemoryContext verb but not its contract

fixable · level 2, rubric item 12 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

An extension author knows that MemoryContextReset() frees what the context holds, and calls ResourceOwnerReset() on an owner that still holds buffer pins or locks. In a build without assertions, the owner is reused and those resources are no longer tracked. The leak shows up later as a pin or lock that is never released.

Message #5 names the function by analogy with MemoryContext. But the README hunk says all resources must be released first, so the verb means something else here. The material does not say whether the function errors, asserts, or does nothing when the owner still holds resources, or when release has not finished. For a public API, a committer would likely ask for a name or a check that makes misuse fail loudly in production builds.

Thread: [#4 (Andres Freund, 2026-11-02)](https://www.postgresql.org/message-id/duhzrnzhykvs2dzmzejrlqtceto7ptj5r7rtm2ues4vuctsnzm%40h7dx6bovzax4), [#5 (Jingtang Zhang, 2026-11-03)](https://www.postgresql.org/message-id/CAPsk3_As%3D%2BDSX3t_WfzWJXHESyz%2B0%3DF%3DbbZ9Q-i4VTykq5_96A%40mail.gmail.com)

**Question:** What does ResourceOwnerReset() do in a build without assertions when the owner or a child still holds resources or has not finished all three release phases?

Counter-check: held. The material does not say what ResourceOwnerReset() checks when resources are still held. A demand for a production-build check goes beyond existing practice, because ResourceOwnerDelete() has the same precondition and enforces it with assertions only.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v4 adds the exported function ResourceOwnerReset() to src/include/utils/resowner.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable and v1 said "Backpatch-through: 17", but v4 adds a public function. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A public reset operation in resowner.c must stay correct as the owner's state changes. |
| performance on the hot path | **yes** | ResourceOwnerSort() and bulk release now clear hash slots for every owner, not only auxiliary ones. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which part of v4 goes to stable branches, and can the back-branch fix work without adding ResourceOwnerReset() to resowner.h?
2. What does ResourceOwnerReset() do in a build without assertions when the owner or a child still holds resources or has not finished all three release phases?
3. How does clearing during sort and bulk release perform for transactions that hold many hashed resources of several kinds?
4. How does ResourceOwnerReset() fit with Rahila Syed's proposal to release all three phases in one call?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund asked in message #4 for the generic helper and for clearing in ResourceOwnerSort(). The owner must know resowner.c and the auxiliary-process cleanup path.
- **Liability:** The owner signs for changed release behavior in every resource owner on all branches that get the fix, and for a new public C function that extensions may call from then on.
- **Author follow-up:** The author posted v2 the same day as the first review and reached v4 with benchmarks within the thread, which suggests continued engagement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 12 KiB and local to resowner.c and its test module. It fixes a double release or hang in reused auxiliary owners, and the remaining points are a back-branch plan and the reset precondition check.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Every new resowner field must also be handled in ResourceOwnerReset. Refuted (already handled): The reuse code exists today in ReleaseAuxProcessResources(), so v4 adds no new place to maintain. It moves that code into the generic resowner code, as Andres Freund asked in message #4. The assertion question survives in standards-compat-2.
- `maintenance-fit-2` Stable-branch fix differs from the master API, with no plan stated. Refuted (not specific): Duplicate of standards-compat-1, which asks the same back-branch question with a more concrete failure scenario.
- `already-exists-1` ResourceOwnerReset() duplicates delete and re-create of the owner. Refuted (already handled): Auxiliary processes already reuse their owner today, so v4 exposes an existing operation and adds no second way. Andres Freund asked for the generic helper in message #4. The back-branch part is covered by standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] and [P2] are the earlier ResourceOwner refactoring that added the hash store, and they were returned only because the patches stopped applying, which does not apply here. Message #6 points to a separate proposal to release all three phases in one call, a related API change rather than an earlier attempt at owner reset.
- standards-compat: 2 objection(s). The patch has no SQL, GUC, catalog, pg_dump or psql surface. The two objections above cover the only compatibility contracts I found: the stable-branch ABI and the C API naming.
- replication-ondisk: 0 objection(s). The proposal changes only in-memory, per-backend resource owner state: no on-disk format, WAL, catalog, replication or wire-protocol changes. Adding ResourceOwnerReset() to resowner.h adds a symbol and does not change any existing signature, and the ResourceOwner struct is private to resowner.c, so a back-branch version would not break extension ABI.
- security: 0 objection(s). The proposal changes internal resource owner bookkeeping and adds a C-level reset function. It adds no SQL-callable surface outside the test module, no privilege, no new data exposure and no path for unprivileged users. The bug it fixes involves double release of resources, so it improves safety more than it weakens it.
- maintenance-fit: 2 objection(s). The core fix is local to resowner.c. Andres Freund, who knows this area, shaped the design, and the measured hot-path cost for non-users is small. That lowers the fit concerns.
- already-exists: 1 objection(s). The bug fix itself, clearing stale slots in ResourceOwnerSort() and bulk release, has no existing alternative in core. Only the new public Reset API overlaps with delete and re-create.
- demand: 0 objection(s). This is a bug fix for a real defect: a reused auxiliary owner can release resources twice or hang in ResourceOwnerAddToHash() (messages #1 and #3), and nobody disputed that. The new ResourceOwnerReset() has one caller today, but a committer who knows the area asked for a generic helper (message #4), so its demand is a design question for other angles.

### Precedents retrieved

- P1: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/2834/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P2: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/3364/) (cf, Returned with feedback, 2022-02-07) - used in the review; reason summarised from its thread
- P3: [Avoid use of TopMemoryContext for resource owner cleanup in portals](https://commitfest.postgresql.org/patch/6603/) (cf, Returned with feedback, 2026-07-07) - judged not relevant; reason summarised from its thread
- P4: [Fix off-by-one in logical slot resource retention](https://commitfest.postgresql.org/patch/1084/) (cf, Returned with feedback, 2017-10-01) - judged not relevant; reason summarised from its thread
- P5: [base backup client as auxiliary backend process](https://commitfest.postgresql.org/patch/2192/) (cf, Returned with feedback, 2020-02-28) - judged not relevant; reason summarised from its thread
- P6: [CREATEROLE and role ownership hierarchies](https://commitfest.postgresql.org/patch/3506/) (cf, Returned with feedback, 2022-04-12) - judged not relevant; reason summarised from its thread
- P7: [Allow cluster owner to bypass authentication](https://commitfest.postgresql.org/patch/2244/) (cf, Returned with feedback, 2020-04-06) - judged not relevant; reason summarised from its thread
- P8: [Safer hash table initialization macro](https://commitfest.postgresql.org/patch/6276/) (cf, Withdrawn, 2026-03-09) - judged not relevant; reason summarised from its thread
- P9: [Reuse data from readRecordBuf in XLogDecodeNextRecord](https://commitfest.postgresql.org/patch/6343/) (cf, Withdrawn, 2026-02-09) - judged not relevant
- P10: [Revert "Introduce CompactAttribute array in TupleDesc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4171c44c9b791da3c00386dc6d8e6b1842e3036b) (revert, Reverted, 2024-12-03) - judged not relevant
- P11: [This adds a new GUC parameter that automatically rolls back prepared transactions that have remained unresolved beyond the configured timeout. The cleanup is performed by the checkpointer process.](https://commitfest.postgresql.org/patch/6611/) (cf, Rejected, 2026-07-01) - judged not relevant
- P12: [Improve compactify_tuples and PageRepairFragmentation](https://commitfest.postgresql.org/patch/1138/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P13: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P14: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant
- P15: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant
- P16: [Revert "Refactor ChangeVarNodesExtended() using the custom callback"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2782f3b8454676c9df408aa0bc960efca0f3eb22) (revert, Reverted, 2025-05-03) - judged not relevant
- P17: [Lazy hash table for snapshot's "xid in progress"](https://commitfest.postgresql.org/patch/1288/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P18: [Speed up LockReleaseAll](https://commitfest.postgresql.org/patch/3220/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P19: [(Duplicate of CF entry /48/4866/) Table Access Method Enhancements](https://commitfest.postgresql.org/patch/4958/) (cf, Rejected, 2024-07-23) - judged not relevant
- P20: [Compression dictionaries](https://commitfest.postgresql.org/patch/3626/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P21: [pg_resetsysid](https://commitfest.postgresql.org/patch/268/) (cf, Returned with feedback, 2015-07-26) - judged not relevant
- P22: [Dynamic result sets from procedures](https://commitfest.postgresql.org/patch/2911/) (cf, Withdrawn, 2023-10-30) - judged not relevant
- P23: [Revert gmake/make release note changes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=993c3961a4166a766c9b0a67701e9c82432550cc) (revert, Reverted, 2014-02-12) - judged not relevant
- P24: [Speed up releasing of locks](https://commitfest.postgresql.org/patch/3501/) (cf, Withdrawn, 2024-05-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 126 s; cost reported by the CLI: $1.79
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 70 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:01:12.859Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
