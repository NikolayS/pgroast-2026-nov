# Design pre-review: Use maintenance_io_concurrency for parallel index builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7280](https://commitfest.postgresql.org/patch/7280/); [thread](https://www.postgresql.org/message-id/flat/CAPsk3_BaXgiA3c%3DGR63kU6O478MWggYtBCx3vWnW6d87F3%3D0nA%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal: these scans do maintenance work and reuse an existing flag and setting. Two fixable points remain. standards-compat-1 asks why a change in prefetch depth belongs in stable branches, and which ones. maintenance-fit-2 asks for a rule on which scans pass SO_MAINTENANCE and why the concurrent validation scan is left out. A committer can settle both, most likely by applying to master only.

Not checked: Without the code, nobody can confirm that the tableam.h change is ABI-compatible on stable branches or that every maintenance heap scan now passes the flag.

## Prior attempts

None found before 2026-09-08.

## Objections a reviewer is likely to raise

### 1. Backpatching changes prefetch depth for existing installations on a minor upgrade

A DBA raised effective_io_concurrency for fast storage and left maintenance_io_concurrency at its default. After a minor upgrade, create index, VACUUM FULL and CLUSTER prefetch less, and nightly maintenance windows run longer. The DBA learns of it only when jobs overrun, with no release note telling them which setting to change.

The CommitFest entry targets "stable", but none of the five messages argues for a backpatch or says which branches. The patch moves which setting controls I/O for common maintenance commands, so tuned systems see a change they did not ask for in a minor release. A committer would likely treat this as a master-only behaviour change unless the docs already promise the new behaviour. REPACK also appears in v4, so a backpatched version would differ by branch and still needs to be defined.

**Question:** Which branches should get this, and why is a changed I/O setting for create index and VACUUM FULL acceptable in a minor release for already tuned systems?

<sub>fixable, level 1b, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAPsk3_BaXgiA3c%3DGR63kU6O478MWggYtBCx3vWnW6d87F3%3D0nA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAPsk3_ASDKa-3fhMTeXqPoa%2BEV35S26_%2BP7uXN_75F_jcbzu0A%40mail.gmail.com); `standards-compat-1`</sub>

### 2. No stated rule for which scans count as maintenance

A developer later adds a heap scan to a maintenance command, or a table AM implements index_build_range_scan, without passing SO_MAINTENANCE. That scan silently uses effective_io_concurrency, and nothing fails. Users see one command follow two settings and report it as a new bug.

Coverage is opt-in per call site. v4 shows this, because the copy_for_cluster scan was found weeks after v3 (message #5). Create index concurrently would also still mix two settings in one command, since the validation scan is left unchanged without a stated reason. A committer would likely ask for a written rule, in the docs or near the flag, that says which scans must pass it.

**Question:** What rule decides which scans must pass SO_MAINTENANCE, and why does the concurrent index validation scan fall outside it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/097AEAB9-FD3E-4B58-922D-8F5AFC06A709%40gmail.com), [#5](https://www.postgresql.org/message-id/CAPsk3_ASDKa-3fhMTeXqPoa%2BEV35S26_%2BP7uXN_75F_jcbzu0A%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Which stable branches should get this change, and why is a different prefetch depth for create index and VACUUM FULL acceptable in a minor release?
2. What rule decides which scans must pass SO_MAINTENANCE, and why does the heap validation scan of concurrent index creation fall outside it?
3. Does the tableam.h change keep struct layout and flag values compatible for table AM extensions compiled against current stable branches?
4. What is the measured effect on create index and VACUUM FULL time when maintenance_io_concurrency and effective_io_concurrency differ?
5. Are non-heap table AMs expected to honour SO_MAINTENANCE in their own index build and rewrite scans?

## Touch points

- **ABI / extension API:** The patch changes src/include/access/tableam.h, which out-of-tree table AMs compile against, and the entry targets stable branches.
- **new GUC:** No new GUC, but maintenance_io_concurrency gains control of create index, BRIN summarization, VACUUM FULL, CLUSTER and REPACK heap scans.
- **backpatchability:** The CommitFest entry is tagged Bugfix with target version stable, and REPACK does not exist in all branches.
- **maintenance burden ("who maintains this in 10 years"):** Each maintenance scan must pass SO_MAINTENANCE itself, and nothing fails when a new call site omits it.
- **performance on the hot path:** The patch changes the prefetch depth of heap scans in index builds and table rewrites, with no benchmark shown.

## Who would commit it, and when

- Owner: silent. Someone who knows the read stream code and the table AM scan interface would have to own it; no committer has spoken in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is 5.7 KB, reuses an existing flag and setting, and helps administrators who tune maintenance I/O separately. The open points are the backpatch scope and a stated rule for which scans pass the flag.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
