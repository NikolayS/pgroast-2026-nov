# Design pre-review: GIN index fast list search may become un-interruptible for long time

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7538](https://commitfest.postgresql.org/patch/7538/); [thread](https://www.postgresql.org/message-id/flat/CALdSSPisYC-zHiWJU36AP-zrPN-im-gEZrK-QD8TPn3Vb8xMvw%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated with a reproducer, and two independent production reports show demand. The surviving objections, maintenance-fit-1 and maintenance-fit-2, are fixable design points: v2 adds no interrupt check for a long pending list with few query entries, and the 64-entry cutoff has no measurements. Neither would stop a committer from reading on. v2-0002 changes when the consistent functions run relative to the lock, which needs a reviewer who knows GIN.

Not checked: The v2 patches and message #4 were not shown, so this rests on the brief's summary of v2. The locking in v2-0002 and the mapping of sorted entries to consistent-function arguments could not be checked without the code.

## Prior attempts

None found before 2026-07-04.

## Objections a reviewer is likely to raise

### 1. Speedup leaves the uncancellable loop in place for other workloads

A site raises gin_pending_list_limit so the pending list holds millions of rows, then runs a query with 50 entries. The query takes the old path, holds the pending list buffer lock with interrupts held off, and again ignores pg_cancel_backend and statement_timeout. The DBA finds out during an incident, after the bug was believed fixed.

The bug report in message #1 is that the scan cannot be interrupted while it holds the buffer lock. v2-0001 in message #4 makes one input shape faster but adds no interrupt check, and queries below 64 entries keep the old path. The time spent still grows with pending list length, so the fix depends on the workload, and a later regression in the fast path would bring the bug back without any test failing.

**Question:** How does v2 keep a scan cancellable when the pending list is very long but the query has few entries, or when the fast path is not used?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CALdSSPisYC-zHiWJU36AP-zrPN-im-gEZrK-QD8TPn3Vb8xMvw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/f7eeaba4-fa1d-4cfd-ab73-44ba393937b9%40iki.fi), [#4](https://www.postgresql.org/message-id/CAGRkXqTAf2ghC9hyc_NLx5oA0PP%3Dv3L8S8LMQRcDxbr9GJ6niQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Two lookup paths behind an unexplained 64-entry cutoff

A later patch changes how pending list rows are matched, for example for partial match or a new strategy, and updates only the linear path. Most regression queries use few entries, so the sorted path is rarely run. Users with large arrays get wrong or missing rows, and the bug shows up only in the field.

Message #4 keeps the old per-key lookup below 64 entries and adds a sorted binary-search path above it. Both paths must return the same matches for every opclass, and nothing fails loudly if they drift apart. The material does not say how 64 was chosen or what small queries cost on the sorted path, so a committer cannot tell whether one path would be enough.

**Question:** What measurements justify keeping two lookup paths, and why would the sorted path alone not be acceptable for queries with few entries?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/CAGRkXqTAf2ghC9hyc_NLx5oA0PP%3Dv3L8S8LMQRcDxbr9GJ6niQ%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. How does v2 keep a scan cancellable when the pending list is very long but the query has fewer than 64 entries?
2. What measurements led to the cutoff of 64 entries, and how do queries with few entries perform on the sorted path?
3. How does v2-0002 keep results correct when the consistent functions run after the pending list lock is released?
4. Is v2 meant for master only, and what change, if any, is proposed for back branches?
5. Is the v1-0002 change that moves the CFI in startScanKey() still part of the plan?

## Touch points

- **security / privileges:** Any user with SELECT on the table can start a scan that ignores cancel and statement_timeout; the patch narrows that.
- **backpatchability:** The entry is tagged Bugfix, Kirill wants a back-patchable fix, and Heikki says the sorting approach is not backpatchable.
- **maintenance burden ("who maintains this in 10 years"):** v2 keeps the old lookup below 64 entries and adds a sorted binary-search path above it in ginget.c.
- **performance on the hot path:** The patches change the pending list scan in ginget.c, which every GIN scan of an index with pending entries runs.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas reviewed v1 in message #2 and suggested the sorting approach; he has not yet replied to v2. The owner needs to know GIN pending list scans and buffer locking.
- Priority: proceed; audience a sizeable minority; review cost medium. Two production reports describe backends that ignore cancel and statement_timeout for minutes, and the change stays inside ginget.c with no user-visible surface. Review cost is medium because the sorted path and the lock change in v2-0002 need a careful read by someone who knows GIN.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
