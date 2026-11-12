# Design pre-review: Vacuum statistics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5012](https://commitfest.postgresql.org/patch/5012/); [thread](https://www.postgresql.org/message-id/flat/bee34abe-fe87-49b3-8523-8ae19e641ccf%40yandex.ru), 161 of 161 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Patches 0001-0004 draw no surviving objection, and the thread shows demand for them. The hook and contrib module in 0005-0008 fail at level 1b. Two reviewers asked which vacuum settings the counters would tune, and message #151 defers the answer (demand-1). The material also does not say why the storage module must ship in contrib (already-exists-2), and it cites no field report (demand-2).

Not checked: Five level 2 objections survived but were not evaluated because level 1b fails. Without the code, the GucContext of vacuum_statistics.enabled, hook chaining with object_access_hook, and the parallel accounting could not be checked.

## Prior attempts

- [Add last_vacuum_index_scans in pg_stat_all_tables](https://www.postgresql.org/message-id/flat/ce4a6ee93d78e2c4dafe4c1e515b342b@oss.nttdata.com) (related; Withdrawn, 2023-01-31). "No other stats counter works that way; they are all additive." ([message](https://www.postgresql.org/message-id/20220708164052.bp62uhma56v7u7n5%40alvherre.pgsql)) Different now: P1 kept only the last vacuum's index-scan count in one column. This proposal keeps additive cumulative counters, which answers the recorded objection, and puts most detail in an extension.

## Objections a reviewer is likely to raise

### 1. No vacuum setting is named that the extension statistics would tune

A DBA loads ext_vacuum_statistics and sees about 30 cumulative counters per table and index. No documented step leads from any counter to a change in autovacuum_vacuum_cost_limit, maintenance_work_mem or another setting, so the DBA watches the numbers and makes no tuning decision. The project then maintains a core hook and a contrib module that does not change what users do.

Amit asked in message #105 how these statistics would be used to tune vacuum, and Bharath repeated the question in message #150. In message #151 the author defers the answer and says it will come later. The stated goal in message #1 is tuning, so the case for 0005-0008 depends on that missing answer.

**Question:** For three or four counters in 0005-0008, which vacuum setting does each one change, and what does a DBA see before and after the change?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/bee34abe-fe87-49b3-8523-8ae19e641ccf%40yandex.ru), [#105](https://www.postgresql.org/message-id/CAA4eK1JOUn%2BEqWSfRgKfgBZOXT7Q2dw2enmSZZgOhoMOFwopPA%40mail.gmail.com), [#150](https://www.postgresql.org/message-id/CALj2ACWoOFfc%2Bx4XEqL9RL2zoU%2BaSXiVbX2kopPQbUiBN%3D_1%2BA%40mail.gmail.com), [#151](https://www.postgresql.org/message-id/15a3a585-b55b-458e-be99-858238427329%40yandex.ru); `demand-1`</sub>

### 2. Once the hook exists, the storage module need not be in contrib

The project commits ext_vacuum_statistics with its views, reset functions, GUC and custom stats kinds, and then owns fixes to its parallel accounting, OID-reuse handling and privileges. Messages #152-#158 show such bugs found late. If someone starts the server once without the module loaded, all cumulative statistics are reset, and users report that to the project.

Bharath (message #150) and Amit (message #105) proposed core hooks with an external module, along the lines of pg_stat_statements, and the author has kept the module on GitHub since message #114. With set_report_vacuum_hook in core, the proposal does not say the module needs contrib for catalog access, grammar or performance. Custom cumulative statistics work out of tree, so contrib placement mainly moves the maintenance burden onto the project.

**Question:** What does ext_vacuum_statistics gain from contrib that an out-of-tree module using set_report_vacuum_hook cannot have?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#105](https://www.postgresql.org/message-id/CAA4eK1JOUn%2BEqWSfRgKfgBZOXT7Q2dw2enmSZZgOhoMOFwopPA%40mail.gmail.com), [#114](https://www.postgresql.org/message-id/79490c8c-2c35-464d-bb3e-3d2ea5aca9ab%40postgrespro.ru), [#150](https://www.postgresql.org/message-id/CALj2ACWoOFfc%2Bx4XEqL9RL2zoU%2BaSXiVbX2kopPQbUiBN%3D_1%2BA%40mail.gmail.com), [#152](https://www.postgresql.org/message-id/CAPpHfdud%3DYAi6Bnmzt4RgPmtshOJw9kHa9PLbfhBAbonJng7_g%40mail.gmail.com), [#157](https://www.postgresql.org/message-id/0234f0e0-ab7f-4397-963c-ae2c207da7e1%40gmail.com); `already-exists-2`</sub>

### 3. Per-run vacuum data already exists in the log; demand for more is not shown

Operators who need per-table vacuum history today set log_autovacuum_min_duration = 0 and aggregate the log lines. 0001-0004 already add totals, delay times and failsafe counts to pg_stat views. The extra buffer, WAL and page breakdowns in 0005-0008 serve a group whose size the thread does not show.

Message #150 maps almost every metric in the series to a field the VACUUM VERBOSE or autovacuum log already prints. The author answers in message #151 that log data is lost to rotation and needs a parser. That is an inconvenience with a working remedy. The interest in the thread comes from reviewers and patch authors (messages #9, #68). No user report from -general or -bugs is cited, and the thread does not show production numbers.

**Question:** Which production cases, outside this thread, could not be diagnosed from the autovacuum log line plus the 0001-0004 columns, and what did those operators need instead?

<sub>fixable, level 1b, demand; messages [#9](https://www.postgresql.org/message-id/CAD21AoDQ3bc%2BebbhMvDy9eZ%3DQBnN-mHTAc0569cbnWXjozx0dw%40mail.gmail.com), [#68](https://www.postgresql.org/message-id/CAA5RZ0sjDx6QFRJJ%2Be%3DorwvLboLWO42f2qF0_HbDq-sMq%3DtW4g%40mail.gmail.com), [#150](https://www.postgresql.org/message-id/CALj2ACWoOFfc%2Bx4XEqL9RL2zoU%2BaSXiVbX2kopPQbUiBN%3D_1%2BA%40mail.gmail.com), [#151](https://www.postgresql.org/message-id/15a3a585-b55b-458e-be99-858238427329%40yandex.ru); `demand-2`</sub>

## Questions for the author

1. For three or four counters in 0005-0008, which vacuum setting does each one lead a DBA to change, and what does the DBA see before and after?
2. What does ext_vacuum_statistics gain from contrib that an out-of-tree module using set_report_vacuum_hook cannot have?
3. Which production cases could not be diagnosed from the autovacuum log line plus the 0001-0004 columns?
4. Can the statistics file reader skip entries of an unregistered custom kind, so that removing the module keeps the built-in counters?
5. Should all vacuum counters include or exclude index work in the table figure, and where is that rule documented for both sets of views?

## Touch points

- **on-disk format / pg_upgrade:** The module writes custom kinds into the statistics file; a start without the module discards all cumulative statistics.
- **catalog changes:** 0002-0004 change system_views.sql and pg_proc.dat and bump catversion for new pg_stat columns.
- **ABI / extension API:** The series adds set_report_vacuum_hook and its report struct, geterrlevel() and visibilitymap_clear_rel().
- **new GUC:** The contrib module adds vacuum_statistics.enabled.
- **security / privileges:** The module adds reset functions restricted to superusers by default, and views that show all relations.
- **maintenance burden ("who maintains this in 10 years"):** Sampling code in vacuumlazy.c and vacuumparallel.c, a core hook and a contrib module all need upkeep.
- **performance on the hot path:** Vacuum samples buffer and WAL usage around the heap run and each index pass, and pgstat entries gain fields.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Reviewers argued the detailed counters belong in a module, and that module could live out of tree.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows lazy vacuum, parallel vacuum and the cumulative statistics system. Reviewers commented on v42-v44, but v47 has no reply.
- Not now, show demand first: A worked mapping from a few of the 0005-0008 counters to specific autovacuum settings, plus field reports of cases that the autovacuum log line and the 0001-0004 columns could not diagnose.
- Priority: needs design rethink; audience a sizeable minority; review cost high. The nine patches touch vacuumlazy.c, vacuumparallel.c, pgstat and a new contrib module, and the design changed direction several times. Patches 0001-0004 could go ahead as their own entry, while 0005-0008 wait for the tuning answer and the decision on contrib placement.

---

[roast-full.md](roast-full.md) has 5 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 4.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
