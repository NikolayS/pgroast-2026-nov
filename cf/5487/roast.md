# Design pre-review: Pull-up subquery if INNER JOIN-ON contains refs to upper-query

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5487](https://commitfest.postgresql.org/patch/5487/); [thread](https://www.postgresql.org/message-id/flat/0b1f670d-b39d-4966-bf32-f0d502ebc564%40postgrespro.ru), 30 of 30 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). Level 1b is where a committer would likely stop. already-exists-1: message #1 shows the WHERE form already gets the semi join, and the thread names no workload that emits the ON form and cannot change it. demand-1: 0002 rests on one Cloudberry discussion, and message #30 measures a query going from about 0.2 ms to 36 ms. Both are requests for evidence, and reviewers report 0001 correct. Confidence is low because the verdict rests on missing demand evidence and only 3 of 30 messages are visible.

Not checked: Without the code, the nullable-side rules of 0001, the row equivalence of the 0002 rewrite and the handling of security-barrier quals could not be checked. The file list and commit messages for v8 to v11 are not in the material.

## Prior attempts

- [Pull up sublink of type 'NOT NOT (expr)'](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com) (related; Rejected, 2018-11-23). "So I'm inclined to reject this patch as probably being a net loss in almost all cases. If you can show any real-world cases where it wins, we could reconsider." ([message](https://www.postgresql.org/message-id/29888.1542319596%40sss.pgh.pa.us)) Different now: P3 targeted a hand-written NOT NOT form. 0001 targets correlation written in an inner join ON clause, but the thread likewise shows no real workload that uses the form.
- [Pulling up direct-correlated ANY_SUBLINK](https://www.postgresql.org/message-id/flat/CAKU4AWoZksNZ4VR-fLTdwmiR91WU8qViDBNQKNwY=7iyo+uV0w@mail.gmail.com) (related; Returned with feedback, 2023-01-31). "This patch seems awfully messy to me. The fact that you're having to duplicate stuff done elsewhere suggests at the least that you've not plugged the code into the best place." ([message](https://www.postgresql.org/message-id/247851.1668293143%40sss.pgh.pa.us)) Different now: P1 pulled up correlated ANY sublinks through LATERAL. 0001 moves ON-clause quals to WHERE and builds a plain semi or anti join.

## Objections a reviewer is likely to raise

### 1. Moving the correlated qual to WHERE already gives the semi join

A user whose ON-correlated EXISTS runs well as a SubPlan upgrades, and 0001 now pulls it up into a slower plan, as in message #12 (about 70 s instead of 68 s). Users who wanted the semi join could already get it by writing the correlation in WHERE.

For inner joins not under an outer join's nullable side, 0001 moves the ON qual to WHERE. A user can make the same rewrite by hand today, and message #1 shows that the WHERE form already gets a Nested Loop Semi Join. The thread shows no field report, ORM output or -general thread where users write or generate the ON form and cannot change it; the only outside motivation, the Cloudberry case in message #29, is for 0002. In [P3], a pull-up of hand-written forms was rejected until real-world cases were shown.

Precedent: [Pull up sublink of type 'NOT NOT (expr)'](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com) (Rejected, 2018-11-23)

**Question:** Which applications or query generators emit correlated EXISTS with the correlation in an inner join ON clause and cannot be changed to put it in WHERE?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/0b1f670d-b39d-4966-bf32-f0d502ebc564%40postgrespro.ru), [#12](https://www.postgresql.org/message-id/87a6fcbf-929a-408a-9471-28be3080254a%40postgrespro.ru), [#29](https://www.postgresql.org/message-id/600857c0-8439-4c9c-8d87-6cb46dfc4406%40yandex.ru); `already-exists-1`</sub>

### 2. 0002 rests on one external bug discussion and slows queries that were fast

A user has a LEFT JOIN whose ON clause has an EXISTS that is fast today as an index-only SubPlan. After upgrade, 0002 picks the LEFT JOIN plus HashAggregate fallback before path generation. The query reads and groups the whole inner table, and the user finds out from a slower production query.

The only stated motivation for 0002 is a Cloudberry bug discussion (message #29), which comes from a fork, and no Postgres user report is cited. Message #30 shows a query going from about 0.2 ms to 36 ms because the rewrite is chosen without costing. The cost falls on users whose queries already run well, while the gain is for an unmeasured group. [P3] was rejected when nobody showed real cases where the pull-up wins, and the same question applies here.

Precedent: [Pull up sublink of type 'NOT NOT (expr)'](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com) (Rejected, 2018-11-23)

**Question:** Which Postgres workloads put a correlated EXISTS in an outer join's ON clause, and how often is the fallback faster there than the SubPlan?

<sub>fixable, level 1b, demand; messages [#29](https://www.postgresql.org/message-id/600857c0-8439-4c9c-8d87-6cb46dfc4406%40yandex.ru), [#30](https://www.postgresql.org/message-id/CAAgWR6ZPLnEEJw67SkF%2BM8VVn-EK0e%3DbGX5BovQraKhUexS%2BCw%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which applications or query generators emit a correlated EXISTS with the correlation in an inner join ON clause and cannot be changed to put it in WHERE?
2. Which Postgres workloads put a correlated EXISTS in an outer join's ON clause, and how often is the 0002 fallback faster there than the SubPlan?
3. How will 0002 keep the SubPlan available when it is cheaper, so the query in message #30 does not get slower after upgrade?
4. What argument, beyond SQLsmith and SQLancer runs, shows that the distinct-key LEFT JOIN with the CASE check returns the same rows as the SubPlan?
5. Does v11 still add the GUC that v5 added, and if so what are its name and default?

## Touch points

- **ABI / extension API:** v5 touches src/include/optimizer/optimizer.h, a header that planner extensions include.
- **new GUC:** v5 touches guc_tables.c and postgresql.conf.sample; the thread does not name the GUC or say if v11 keeps it.
- **planner behaviour change:** More EXISTS and NOT EXISTS sublinks become semi, anti or left joins, so existing plans change.
- **maintenance burden ("who maintains this in 10 years"):** A new jointree walker with nullable-side rules in subselect.c must stay correct through later outer-join changes.
- **performance on the hot path:** 0002 picks its rewrite before path generation; message #30 shows a query about 175 times slower.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A user can already get the semi join by writing the correlation in WHERE, as message #1 shows.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows pull_up_sublinks and the outer-join nullable-side rules in the planner. None has commented in the visible thread or is listed on the CommitFest entry.
- Not now, show demand first: For 0001: real queries, ORM output or user reports that put the correlation in an inner join ON clause and cannot be rewritten, with measured gains. For 0002, demand evidence is not enough alone; it also needs a way to keep the SubPlan when it is cheaper.
- Priority: needs champion; audience unclear; review cost high. Reviewers report 0001 correct after random-query testing, but no committer has engaged in 10 CommitFests and the thread shows no user demand. The v5 patch was about 54 KiB in subselect.c with subtle nullable-side rules, and 0002 still has an open design question from message #30.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
