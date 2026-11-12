# Design pre-review: using index to speedup add not null constraints to a table

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5444](https://commitfest.postgresql.org/patch/5444/); [thread](https://www.postgresql.org/message-id/flat/CACJufxFiW%3D4k1is%3DF1J%3Dr-Cx1RuByXQPUrWB331U47rSnGz%2Bhw%40mail.gmail.com), 22 of 22 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and a committer called the idea great in message #11. The open points are fixable design questions. maintenance-fit-1 asks which check guarantees the index is not rebuilt in the same alter table. maintenance-fit-2 asks for a cost bound when the index holds dead null entries. already-exists-1 stands only as a request to compare with `not valid` plus `validate constraint`.

Not checked: Without the code, this review cannot confirm which indexes the patch excludes or that the phase 3 conditions cover every subcommand that rebuilds an index. The visibility argument in the v16 commit message was also not checked against the code.

## Prior attempts

None found before 2024-12-16.

## Objections a reviewer is likely to raise

### 1. Correctness rests on an implicit list of phase 3 conditions

A later patch adds or changes an alter table subcommand so that an index is rebuilt or changed in the same command, and the fast-path checks are not updated. The index scan then runs on an index that does not match the final table. A not-null constraint is accepted while nulls exist, and users find out only when queries or pg_dump restores fail.

Message #1 and the v1 commit message show that subcommands which trigger index rebuilds, such as SET EXPRESSION, already broke this approach. v4 (message #4) listed "no new indexes" as a condition, but the v16 commit message lists only scan or rewrite, the lock level and virtual generated columns. The only tests are a TAP test that greps a DEBUG1 message (message #18), and none of them fails when a new subcommand changes these conditions.

**Question:** Which single check guarantees the chosen index is not rebuilt or changed in the same alter table, and what test fails if a future subcommand breaks that guarantee?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CACJufxFiW%3D4k1is%3DF1J%3Dr-Cx1RuByXQPUrWB331U47rSnGz%2Bhw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CACJufxEqdgwUTo3xmUR3Bs49dHvD9VZW%2BT1D8sVdBV4-UdDcrg%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/CACJufxE-%2BVSkxMvBRbpM45iex%3D7ws4-km4MvmOBPXtTJ7jjdQA%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CACJufxEkSyhOotK0b3hj_D3c9HO%3DdxGoGxL3Cx4j6enG_t_P6Q%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. No bound or opt-out when the index holds many dead nulls

An operator deletes or updates many rows that had nulls, and vacuum has not run. They then run set not null on a large table whose cache is cold. Each dead index entry needs a random heap fetch under AccessExclusiveLock, so the command can run longer than the sequential scan it replaced, and no setting turns the fast path off.

The v1 commit message reports that the patch is slower than master when null entries are bloated (1080 ms against 971 ms), and that test used a warm table with only a million rows. In bench.txt the first run took about 480 ms with and without the index, so the gains shown come from repeated runs on a warm cache. The patch picks the index path whenever an index exists and does not look at cost or bloat first.

**Question:** How slow is the index path on a cold cache when the index holds many dead null entries, and why is choosing it with no cost check safe enough?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/CACJufxFiW%3D4k1is%3DF1J%3Dr-Cx1RuByXQPUrWB331U47rSnGz%2Bhw%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CACJufxEkSyhOotK0b3hj_D3c9HO%3DdxGoGxL3Cx4j6enG_t_P6Q%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Overlap with the existing not valid plus validate constraint path

A DBA with a large table runs `alter table ... add constraint ... not null` and expects the index fast path. If the index is bloated, the cache is cold, or a second clause forces a scan, the full scan runs under AccessExclusiveLock. The existing path would have avoided that lock: add a not valid constraint, then run validate constraint, which takes a weaker lock.

Postgres already supports adding a not-null constraint as `not valid` and validating it later with `validate constraint` under ShareUpdateExclusiveLock, so writes continue during the scan. Older releases had the same option through a validated `check (col is not null)` constraint, which lets `set not null` skip the scan. The proposal makes the one-step command faster but does not compare itself with these paths. In bench.txt the first run is about the same with and without the index (about 480 ms), so the gain may depend on a warm cache.

**Question:** For which users is `not valid` plus `validate constraint` not enough, and does the index path still help on a cold cache?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CACJufxFiW%3D4k1is%3DF1J%3Dr-Cx1RuByXQPUrWB331U47rSnGz%2Bhw%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CACJufxEkSyhOotK0b3hj_D3c9HO%3DdxGoGxL3Cx4j6enG_t_P6Q%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Which single check guarantees that the chosen index is not rebuilt or changed in the same alter table, and which test fails if a future subcommand breaks it?
2. How slow is the index scan on a cold cache when the index holds many dead null entries, compared with the table scan?
3. Why is it safe to choose the index scan with no cost or bloat check, given the 1080 ms against 971 ms result in the v1 commit message?
4. For which users is `not valid` plus `validate constraint` not enough, so that the one-step fast path is needed?
5. Does v16 use the index scan for partitioned tables, and does it exclude partial, invalid and expression indexes?

## Touch points

- **planner behaviour change:** see objection maintenance-fit-2: No bound or opt-out when the index holds many dead nulls
- **maintenance burden ("who maintains this in 10 years"):** The fast path in tablecmds.c depends on a list of phase 3 conditions that later alter table subcommands must keep true.
- **performance on the hot path:** The index scan runs under AccessExclusiveLock and was slower than the table scan when the index held dead null entries.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** `not valid` plus `validate constraint` already avoids the long scan under AccessExclusiveLock.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows alter table phase 3 and not-null constraints. Álvaro Herrera reviewed it in the thread but removed himself as committer in April 2026.
- Priority: needs champion; audience niche; review cost medium. The patch helps only when the column leads a valid btree index and no scan or rewrite is pending. v16 touches only tablecmds.c and adds no user-visible surface, but it has had no committer since April 2026 and no visible reply to v14 to v16.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
