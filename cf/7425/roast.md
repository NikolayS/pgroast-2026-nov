# Design pre-review: Unlocked path for GetSnapshotDataReuse

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7425](https://commitfest.postgresql.org/patch/7425/); [thread](https://www.postgresql.org/message-id/flat/CAEze2Wjj5LPNpuq6fNy-wv552wbk61dGRJ52NdV%2BWddq0Uf33A%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and a committer who knows this code supports it in message #3. Three fixable design points remain: the hot standby argument for 0003 (replication-ondisk-1), the undocumented pairing of a plain atomic write with the barrier in LWLockRelease (maintenance-fit-1), and the missing benchmarks (maintenance-fit-2). None of them stops review; each needs sign-off from someone who knows procarray.c.

Not checked: The code was not read, so the barrier placement, the order of the xmin write and the second counter read, and the hot standby paths could not be checked. The safety of 0003 rests on arguments in messages #5 and #6 that this review saw only through the brief and the objections.

## Prior attempts

- [POC: Cache data in GetSnapshotData()](https://www.postgresql.org/message-id/flat/CAD__OugYQmNP5dFkGdSjP48B8WjD0+rtff3iupThnQxTXLEUog@mail.gmail.com) (similar idea; Returned with feedback, 2017-09-29). "As Andres has suggested previously I need a mechanism to identify and avoid caching for such scenarios." ([message](https://www.postgresql.org/message-id/CAD__OujRZEjE5y3vfmmZmSSr3oYGZSHRxwDwF7kyhBHB2BpW_g%40mail.gmail.com)) Different now: P3 shared one snapshot across backends in shared memory and paid for the cache when writes invalidated it. This proposal keeps the existing per-backend cache and only drops the lock from the validity check, with a fallback to the locked path.
- [Move PinBuffer and UnpinBuffer to atomics](https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell) (related; Returned with feedback, 2015-12-16). "Which you made ineffective because PinBuffer() doesn't take a lock anymore. Mutual exclusion through locks only works if all participants take the locks." ([message](https://www.postgresql.org/message-id/20150914154142.GB9666%40alap3.anarazel.de)) Different now: P2 removed a lock that other code still relied on for mutual exclusion. Here only the owning backend writes its xmin, and a second read of the counter replaces the exclusion against committers.
- [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org) (related; Reverted, 2022-05-31). "In other words, Xmin advancement was sped up, but at the cost of corrupting the resulting index." Different now: P11 let horizons ignore a backend's xmin. 0003 installs the xmin without the lock and rechecks the counter, which is a different mechanism.

## Objections a reviewer is likely to raise

### 1. Lockless xmin install in 0003 has no stated argument for hot standby

On a hot standby, a backend with a cached snapshot and no xmin installs that snapshot's old xmin without ProcArrayLock while the startup process replays a prune record. The startup process checks for conflicts, sees no xmin on that backend, and removes the tuples. The backend's counter check passes because no completion was replayed, so the query reads pages missing rows it should see.

The safety argument in message #5 covers only primary-side writers: owner-only writes to xmin, plus transaction completion bumping xactCompletionCount. On a standby, other events advance horizons: prune replay with recovery conflicts, and KnownAssignedXids expiry from running-xacts records for xids that never got an abort record. The lockless path is safe only if every such event either bumps xactCompletionCount or cannot pass an xmin that a valid reused snapshot still needs. Message #6 accepts the reasoning without discussing standby, and the patch adds no rule that future standby code must follow.

Precedent: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org) (Reverted, 2022-05-31)

**Question:** On a hot standby, which replay events can remove data a reused snapshot needs without bumping xactCompletionCount, and why does 0003 still never return wrong results?

<sub>fixable, level 2, WAL / replication / standby; messages [#5](https://www.postgresql.org/message-id/CAEze2WgQsX%3Ds%3DXpmSf7q7-rHhWmMJaSBea7xF%2BOevXE29KaxEg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/asym8dZmZM3vd7AV%40bdtpg); `replication-ondisk-1`</sub>

### 2. Counter correctness depends on an undocumented barrier in LWLockRelease

A later patch moves an xactCompletionCount increment outside the ProcArrayLock exclusive section, for example into group clear or a new completion path, or changes LWLockRelease ordering. The lockless reader can then see a stale count and reuse a snapshot that misses a commit. No test fails; users see wrong visibility results long after release.

Message #6 notes that IncrementXactCompletionCount() uses a plain pg_atomic_write_u64() and relies on the full barrier in the following LWLockRelease(ProcArrayLock), and that no other code in the tree uses this pairing. Message #3 asks for the same pairing to be documented. Today the Assert on LW_EXCLUSIVE guards the lock, but nothing ties the read side's correctness to that release barrier. Each future completion path must preserve an invariant that is visible only in comments.

Precedent: [Move PinBuffer and UnpinBuffer to atomics](https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell) (Returned with feedback, 2015-12-16)

**Question:** What fails loudly, by assertion or by API shape, if a future increment site stops being followed by a barrier-carrying release?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/lgelgkjrkvnrqb2mfbgcx3fnr3cdr6f2tyanfvhsiyfduw5njz%40okoelncmxfmo), [#6](https://www.postgresql.org/message-id/asym8dZmZM3vd7AV%40bdtpg); `maintenance-fit-1`</sub>

### 3. No measurement of the new path, including membarrier reads on ARM

On a weakly ordered platform, every snapshot taken by every backend now does one or two pg_atomic_read_membarrier_u64() calls. A workload with frequent commits fails the counter check and takes the lock anyway, so it pays for both. An OLTP user on ARM finds a regression only after upgrading.

The only numbers are call counts from the regression suite (message #1); in message #5 the author asserts that membarrier reads cost less than the lock acquisition but shows no measurement. Messages #2 and #6 ask for benchmarks, and message #6 proposes a cheaper first read. GetSnapshotData() is on the path of nearly every query, so the cost falls on all users. In [P3], snapshot reuse work stalled on a regression at low client counts and on workloads where commits invalidate the cache before it is reused.

Precedent: [POC: Cache data in GetSnapshotData()](https://www.postgresql.org/message-id/flat/CAD__OugYQmNP5dFkGdSjP48B8WjD0+rtff3iupThnQxTXLEUog@mail.gmail.com) (Returned with feedback, 2017-09-29)

**Question:** What throughput do x86 and ARM show at 1 client and at high client counts, for read-only and commit-heavy pgbench runs?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/CAEze2Wjj5LPNpuq6fNy-wv552wbk61dGRJ52NdV%2BWddq0Uf33A%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/asiCYBUj0RQ6FA5C%40bdtpg), [#5](https://www.postgresql.org/message-id/CAEze2WgQsX%3Ds%3DXpmSf7q7-rHhWmMJaSBea7xF%2BOevXE29KaxEg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/asym8dZmZM3vd7AV%40bdtpg); `maintenance-fit-2`</sub>

## Questions for the author

1. On a hot standby, which replay events can remove data a reused snapshot needs without bumping xactCompletionCount, and why does 0003 still never return wrong results?
2. What fails loudly, by assertion or by API shape, if a future xactCompletionCount increment is no longer followed by a barrier-carrying LWLockRelease?
3. What throughput do x86 and ARM show at 1 client and at high client counts, for read-only and commit-heavy pgbench runs?
4. How often do vacuum horizons move backward with 0003, and which horizon consumers were checked against that?
5. What does moving only xactCompletionCount out of TransamVariables cost committing backends, compared with the full split into varsup.h?

## Touch points

- **WAL / replication / standby:** 0003 installs xmin without ProcArrayLock, and the thread gives no safety argument for hot standby backends.
- **ABI / extension API:** 0001 moves TransamVariablesData to a new varsup.h, and xactCompletionCount becomes an atomic type.
- **maintenance burden ("who maintains this in 10 years"):** The lockless read relies on every xactCompletionCount increment being followed by the barrier in LWLockRelease(ProcArrayLock).
- **performance on the hot path:** GetSnapshotData() runs for nearly every query, and the new path adds membarrier reads of the counter.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund, who worked on the original xactCompletionCount code, reviews the patch in message #3 and says the optimization ought to be done.
- Priority: proceed; audience a sizeable minority; review cost high. The gain goes to busy systems with many backends, and a committer who knows the area is engaged. The patches are small, but lockless reasoning in procarray.c needs careful review, and the memory-ordering rule must be kept by all later completion paths.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
