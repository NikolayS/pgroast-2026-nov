# Design pre-review: Perf: Specialize ExecIndexOnlyScan() like ExecSeqScan()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7468](https://commitfest.postgresql.org/patch/7468/) (Needs review); authors: Matthias Dötsch (brain.exe)
- Thread: [Perf: Specialize ExecIndexOnlyScan() like ExecSeqScan()](https://www.postgresql.org/message-id/flat/3684e94b-3077-4430-94d9-e5411aa5f110%40mdoetsch.de), started 2026-10-14
- Messages read: 7 of 7
- Patch prose read: v3-0001-Add-isolation-test-for-an-index-only-scan-under-E.patch, v3-0002-Specialize-ExecIndexOnlyScan-like-ExecSeqScan.patch, v2-0001-Specialize-ExecIndexOnlyScan-like-ExecSeqScan.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. The only one raised, maintenance-fit-1, was refuted: the EPQ, qual and projection logic stays in the shared ExecScanExtended(), and the patch adds one copy of the committed SeqScan pattern in one file. The problem is stated with measurements, and the cost falls only on nodeIndexonlyscan.c. Confidence is medium because the code was not read and wall-clock gains depend on code layout.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody here checked that each variant evaluates the quals and projection it must, or that every pg_assume() holds after the runtime-key setup. The new EPQ isolation test's coverage of the variant selection was also not checked.

## The proposal, restated

**Problem.** Each tuple of an index-only scan goes through the out-of-line ExecScan(). It pays for an indirect call to IndexOnlyNext() and for runtime EPQ, qual and projection checks. The author saw this cost in perf profiles of a workload heavy in index-only scans.

**Proposed design.** The patch adds ExecIndexOnlyScan variants for no qual or projection, qual, projection, both, and EPQ, and ExecInitIndexOnlyScan() picks one. Each variant passes const NULLs to the inline ExecScanExtended(), and IndexOnlyNext() becomes pg_always_inline; the EPQ variant still calls ExecScan(). A separate patch adds an isolation test for an index-only scan under EPQ.

**What the author claims.**
- Callgrind shows about 5 to 6% fewer executed instructions on a 1M-row index-only range scan, with and without qual or projection.
- Wall-clock time drops by about 3 to 11%, but code layout alone moves it by several percent.
- With LTO the patch still saves about 5% instructions on top of the LTO gain.
- With JIT the instruction savings are the same as without JIT.
- For a nested loop with an inner index-only scan the gain is under 1% instructions, within noise on wall-clock.

**Thread state.** Andres Freund argued that LTO, PGO and batching cannot replace the specialization, and Haibo Yan withdrew his concerns after his own tests and contributed the EPQ test. David Geier supports the change and asks to extend it to other scan nodes, possibly with a shared abstraction.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-14; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

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
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds five ExecIndexOnlyScan variants and an ordering rule between runtime-key setup and pg_assume() in nodeIndexonlyscan.c. |
| performance on the hot path | **yes** | The patch changes the per-tuple path of every index-only scan and grows its code from about 0.6 kB to 3.5 kB. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the instruction and wall-clock gain hold with clang and on ARM, since the benchmarks cover only gcc on x86-64?
2. Which function alignment represents real builds, given that the qual-case latency gain vanished with 64-byte alignment in Haibo Yan's test?
3. Is this patch meant to stand alone, or will IndexScan and BitmapHeapScan follow, and if so with a shared helper as message #7 asks?
4. What is the gain on the production workload that showed ExecScan() in the perf profiles?
5. Who should the author line of the EPQ isolation test in v3-0001 name?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund defended the specialization against the LTO, PGO and batching concerns in message #3; an executor committer would have to own the commit.
- **Liability:** The owner signs for correct variant selection in ExecInitIndexOnlyScan(), for pg_assume() calls that stay true, and for the EPQ path, for as long as the SeqScan pattern exists.
- **Author follow-up:** The author offered to write the EPQ isolation test in message #1 and posted v3 with further measurements; this is a first patch, so there is no longer record.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Index-only scans are common, but the gain is about 5% of instructions on long range scans and under 1% for a nested loop with an inner index-only scan. The patch is 7.8 KB in one executor file, copies committed SeqScan practice, and adds an EPQ isolation test.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Per-node copies of scan variants before a shared pattern is agreed. Refuted (no failure scenario): The EPQ, qual and projection logic stays in the shared ExecScanExtended(), and a false pg_assume() trips an assertion in cassert builds. The scenario depends on later patches for other nodes; this patch adds one copy of the committed SeqScan pattern, and a missed ordering rule only loses the gain.

### Review angles

- prior-attempts: 0 objection(s). No precedent found that is the same idea; message #1 cites the committed SeqScan specialization as its model. [P3] and [P4] failed for lack of benefit or evidence, but this proposal shows callgrind gains on two machines; [P1] has no recorded reason.
- standards-compat: 0 objection(s). The patch is internal to nodeIndexonlyscan.c and adds no syntax, GUC, catalog object, psql change or client-visible behaviour, so nothing needs a pg_dump round-trip. Query results and plans stay the same, and EPQ semantics are kept through ExecScan(), with a new isolation test in message #4.
- replication-ondisk: 0 objection(s). The patch changes only executor code in nodeIndexonlyscan.c and adds an isolation test. It touches no on-disk format, WAL record, catalog, wire protocol or replication path. The new ExecIndexOnlyScan variants are static functions picked at node init, so extension ABI is not affected. ExecIndexOnlyScan reached through the node's ExecProcNode pointer is executor-internal, not an extension API.
- security: 0 objection(s). The proposal changes only how the executor dispatches index-only scan tuples. It adds no SQL, GUC, privilege, catalog entry or view, and it does not change what any role can run or see. RLS quals and security barrier quals are still evaluated as node quals in the qual variants, and EPQ still goes through ExecScan(), so no privilege or exposure aspect is visible. Whether a variant skips a qual it should evaluate can only be checked in the code.
- maintenance-fit: 1 objection(s). The objection is minor. The design copies committed SeqScan practice, a committer who knows the executor took part in the thread (message #3), the cost falls only on the patched node, and no change to an API or on-disk format is involved.
- already-exists: 0 objection(s). No existing feature or extension meets this need. The patch reuses ExecScanExtended() and the SeqScan pattern rather than adding a new mechanism. Message #5 shows LTO and JIT keep the ExecScan() overhead, and executor dispatch cannot move to an extension or hook. Whether the per-node boilerplate should be shared, as message #7 asks, is a design question for another angle.
- demand: 0 objection(s). Demand looks real. Index-only scans are a common plan node; the author found the cost in perf profiles of a production workload (message #1), Andres Freund and David Geier support the change (messages #3, #7), and Haibo Yan withdrew his concerns (message #6). The patch copies the accepted ExecSeqScan specialization, and its cost stays in nodeIndexonlyscan.c. The gain is under 1% for nested loops with an inner index-only scan, which may be the most common use (message #5). That narrows the benefit to long range scans, but a committer would likely not reject a cheap, contained change for that reason. The material does not show the gain on the author's own production workload.

### Precedents retrieved

- P1: [nbtree performance improvements through specialization on key shape](https://commitfest.postgresql.org/patch/3672/) (cf, Withdrawn, 2026-10-01) - used in the review
- P2: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P3: [Minor refactor of ExecScanExtended()](https://commitfest.postgresql.org/patch/6255/) (cf, Withdrawn, 2026-04-01) - used in the review; reason summarised from its thread
- P4: [Preventing indirection for IndexPageGetOpaque for known-size page special areas](https://commitfest.postgresql.org/patch/3543/) (cf, Rejected, 2022-11-30) - used in the review; reason summarised from its thread
- P5: [Revert "Specialize MemoryContextMemAllocated()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2fd6a44ad5744905e9ceacef7aaf698f608e6adb) (revert, Reverted, 2020-03-19) - judged not relevant
- P6: [Allow direct lookups of SpecialJoinInfo by ojrelid](https://commitfest.postgresql.org/patch/4313/) (cf, Withdrawn, 2023-09-06) - judged not relevant; reason summarised from its thread
- P7: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P8: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P9: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P10: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant; reason summarised from its thread
- P11: [SQL Inlining Behavior](https://commitfest.postgresql.org/patch/1965/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P12: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P13: [Remove special case which avoided %*s format strings..](https://commitfest.postgresql.org/patch/2666/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P14: [Fix standby may crash when switching-over in certain special cases](https://commitfest.postgresql.org/patch/5199/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P15: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P16: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P17: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant
- P18: [indirect indexes](https://commitfest.postgresql.org/patch/874/) (cf, Returned with feedback, 2017-04-01) - judged not relevant
- P19: [Document atthasmissing default optimization avoids verification table scan](https://commitfest.postgresql.org/patch/3329/) (cf, Withdrawn, 2022-03-29) - judged not relevant
- P20: [Add inline comments to the pg_hba_file_rules view](https://commitfest.postgresql.org/patch/4546/) (cf, Withdrawn, 2023-10-10) - judged not relevant
- P21: [minimizing pg_stat_statements.track='none' performance overhead](https://commitfest.postgresql.org/patch/2092/) (cf, Withdrawn, 2019-04-15) - judged not relevant
- P22: [Redundant qualifier elimination](https://commitfest.postgresql.org/patch/7127/) (cf, Returned with feedback, 2026-08-14) - judged not relevant
- P23: [Avoid multiple calls to memcpy (src/backend/access/index/genam.c)](https://commitfest.postgresql.org/patch/6586/) (cf, Withdrawn, 2026-03-25) - judged not relevant
- P24: [Table AM modifications to accept column projection lists](https://commitfest.postgresql.org/patch/2922/) (cf, Withdrawn, 2021-06-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 85 s; cost reported by the CLI: $1.36
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:58:37.609Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
