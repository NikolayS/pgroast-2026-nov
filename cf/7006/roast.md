# Design pre-review: Add bypass IO to pg_stat_io

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7006](https://commitfest.postgresql.org/patch/7006/); [thread](https://www.postgresql.org/message-id/flat/30e8f366-58b3-b239-c521-422122dd5150%40iki.fi), 55 of 55 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). No surviving objection questions the goal, and message #3 shows a committer asked for this use of bulk_write.c. The two surviving objections, maintenance-fit-1 and standards-compat-1, are fixable design points about what the "bypass" rows cover and what the uncounted cells show. Confidence is low because the pg_stat_io patch has no visible reply, no documentation hunk, and its message was not shown in full.

Not checked: The pg_stat_io patch and its documentation were not read, so the valid combinations for the new context, the row count and the counting cost could not be checked. The list of direct smgr writers outside bulk_write.c also needs the code.

## Prior attempts

None found before 2023-09-19.

## Objections a reviewer is likely to raise

### 1. Counting only in smgr_bulk_flush() leaves other direct smgr writers silently uncounted

A future core change or a table AM extension writes relation pages with smgrextend() or smgrwrite() outside bulk_write.c. Its writes do not appear under "bypass", and no test or assertion notices. A DBA reads the bypass rows as the full picture of IO outside shared buffers and misses that load during an incident.

Message #55 puts the accounting in one bulk_write.c function, while bypass reads and smgrimmedsync() stay uncounted. The context name promises all IO that bypasses shared buffers, but the proposal covers only one caller. The earlier pg_stat_io discussion that message #55 cites pointed to smgr wrappers, which would cover every caller. The proposal does not say why the bulk_write layer is better, or what stops new smgr callers from skipping the counters.

**Question:** Which smgrextend()/smgrwrite() callers stay outside bulk_write.c today, and how will future callers be counted or flagged instead of being missed silently?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/20231119000416.fuyrbo6av7dytmf6%40awork3.anarazel.de), [#55](https://www.postgresql.org/message-id/789656F9-C573-474B-8C44-56FCCC16A8D3%40yandex-team.ru); `maintenance-fit-1`</sub>

### 2. Bypass rows would report reads and fsyncs that do happen as untracked

An operator runs ALTER TABLE SET TABLESPACE, which reads through smgrread() in RelationCopyStorage, or a CREATE INDEX that hits smgrimmedsync(). The bypass row in pg_stat_io then shows NULL or 0 for reads and fsyncs, and dashboards trust it. After the planned follow-up patches the same columns fill in, so numbers jump between releases with no change in workload.

pg_stat_io uses NULL for combinations that cannot occur and 0 for counted-but-none. Message #55 says bypass reads and smgrimmedsync() do occur but stay uncounted for now, so either value is wrong for this release. The proposal shows no documentation hunk that defines the valid backend type, object and operation combinations for the new context. Changing what a released pg_stat_io row means in a later release is a cost that monitoring tools would have to handle.

**Question:** What will the bypass row show for reads and fsyncs in this release, and how will the documentation keep that meaning stable once later patches count them?

<sub>fixable, level 2, design; messages [#55](https://www.postgresql.org/message-id/789656F9-C573-474B-8C44-56FCCC16A8D3%40yandex-team.ru); `standards-compat-1`</sub>

## Questions for the author

1. Which smgrextend() and smgrwrite() callers stay outside bulk_write.c today, and will their writes appear under "bypass" or stay uncounted?
2. What will the bypass rows show for reads and fsyncs in this release: NULL, 0, or no such column value?
3. How will the documentation define the bypass rows so that their meaning stays stable once later patches count reads and smgrimmedsync()?
4. How many pg_stat_io rows does the new context add, and for which backend type and object combinations?
5. What is the measured overhead of counting and timing in smgr_bulk_flush() during a large CREATE INDEX with track_io_timing on?

## Touch points

- **ABI / extension API:** A new IOContext value enlarges the cumulative statistics arrays and changes the enum that extensions compile against.
- **maintenance burden ("who maintains this in 10 years"):** The new io_context value "bypass" becomes a released pg_stat_io meaning that later read and fsync patches must keep stable.
- **performance on the hot path:** The patch adds counting and optional timing in smgr_bulk_flush(), once per batched flush, with no measurement shown.

## Who would commit it, and when

- Owner: silent. Someone who knows pg_stat_io and the smgr and bulk_write.c code would have to own it. Committers took part in the 2023-2024 thread, but none has replied to the July 2026 patch.
- Priority: needs champion; audience a sizeable minority; review cost low. The change is additive, sits in one function, and helps users who monitor IO from index builds and table rewrites. It has waited since July 2026 with no reply and needs a rebase, so it needs a reviewer who knows pg_stat_io.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
