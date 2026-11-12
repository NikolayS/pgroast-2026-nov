# Design pre-review: Perf: Specialize ExecIndexOnlyScan() like ExecSeqScan()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7468](https://commitfest.postgresql.org/patch/7468/); [thread](https://www.postgresql.org/message-id/flat/3684e94b-3077-4430-94d9-e5411aa5f110%40mdoetsch.de), 7 of 7 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. The only one raised, maintenance-fit-1, was refuted: the EPQ, qual and projection logic stays in the shared ExecScanExtended(), and the patch adds one copy of the committed SeqScan pattern in one file. The problem is stated with measurements, and the cost falls only on nodeIndexonlyscan.c. Confidence is medium because the code was not read and wall-clock gains depend on code layout.

Not checked: Without the code, nobody here checked that each variant evaluates the quals and projection it must, or that every pg_assume() holds after the runtime-key setup. The new EPQ isolation test's coverage of the variant selection was also not checked.

## Prior attempts

None found before 2026-10-14.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does the instruction and wall-clock gain hold with clang and on ARM, since the benchmarks cover only gcc on x86-64?
2. Which function alignment represents real builds, given that the qual-case latency gain vanished with 64-byte alignment in Haibo Yan's test?
3. Is this patch meant to stand alone, or will IndexScan and BitmapHeapScan follow, and if so with a shared helper as message #7 asks?
4. What is the gain on the production workload that showed ExecScan() in the perf profiles?
5. Who should the author line of the EPQ isolation test in v3-0001 name?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch adds five ExecIndexOnlyScan variants and an ordering rule between runtime-key setup and pg_assume() in nodeIndexonlyscan.c.
- **performance on the hot path:** The patch changes the per-tuple path of every index-only scan and grows its code from about 0.6 kB to 3.5 kB.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund defended the specialization against the LTO, PGO and batching concerns in message #3; an executor committer would have to own the commit.
- Priority: proceed; audience a sizeable minority; review cost low. Index-only scans are common, but the gain is about 5% of instructions on long range scans and under 1% for a nested loop with an inner index-only scan. The patch is 7.8 KB in one executor file, copies committed SeqScan practice, and adds an EPQ isolation test.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
