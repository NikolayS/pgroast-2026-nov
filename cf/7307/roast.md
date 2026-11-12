# Design pre-review: ANSI SQL proposal: SELECT DISTINCT ON (... ORDER BY ...) and UNION DISTINCT ON (... ORDER BY ...)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7307](https://commitfest.postgresql.org/patch/7307/); [thread](https://www.postgresql.org/message-id/flat/CAMT0RQR_QqLqKdLgh3zYk7yA7oH9RJBVF%2BkZJ607wjJD9WKH9Q%40mail.gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Level 1b fails on two fixable objections. standards-compat-1: message #9 says wider approval of the syntax comes only after v19, and message #4 endorses only the inline ORDER BY, so the UNION form would ship ahead of the committee. demand-1: the thread shows no non-recursive workload that DISTINCT ON over a UNION ALL subquery does not already serve. The inline ORDER BY for SELECT drew no surviving objection.

Not checked: Two level 2 objections survived but were not evaluated: the undocumented recursive limits (standards-compat-2) and the series shape (maintenance-fit-2). The security_barrier pushdown rule and the full set of planner paths that relied on the DISTINCT ON and ORDER BY match cannot be judged without the code.

## Prior attempts

- [Removing useless DISTINCT clauses](https://www.postgresql.org/message-id/flat/CAKJS1f8UMJ137sRuVSnEMDDpa57Q71JuLZi4yLCFMekNYVYqaQ@mail.gmail.com) (related; Returned with feedback, 2018-04-07). "Once I noticed that, I became convinced that this patch actually breaks planning of sort/unique-based DISTINCT: if we have a pkey a,b, but the ORDER BY list is a,c,b, then we will sort by a,c,b but the Unique node will be told it only needs to unique-ify on the first two sort columns." ([message](https://www.postgresql.org/message-id/4779.1523134536%40sss.pgh.pa.us)) Different now: P2 removed DISTINCT columns inside the planner; this proposal lets the user separate DISTINCT ON keys from the outer ORDER BY. Both relax the parser-enforced match between DISTINCT and ORDER BY.

## Objections a reviewer is likely to raise

### 1. New DISTINCT ON spellings ship before the standard committee has reviewed them

Postgres ships `union distinct on (k order by c)` in a release, and users put it in views. The committee later adopts a different spelling or different tie-break rules for subset deduplication. Postgres must then support both forms indefinitely, and portable SQL using the standard form behaves differently from existing Postgres views.

The discussion paper (message #1) is a draft, and message #9 says wider approval will be sought only after v19. The standard already covers per-key selection with QUALIFY and FETCH FIRST ... PARTITIONS BY (message #4), and DuckDB uses USING KEY for the recursive case (message #6), so the committee has other spellings to choose from. Vik Fearing endorsed the inline ORDER BY for SELECT but questioned UNION DISTINCT ON, and that is the part with no precedent at all.

**Question:** Has WG 3 seen the UNION DISTINCT ON form, and would the Postgres design still hold if the committee chose USING KEY or different semantics?

<sub>fixable, level 1b, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/CAMT0RQR_QqLqKdLgh3zYk7yA7oH9RJBVF%2BkZJ607wjJD9WKH9Q%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/e9b65da3-9326-4951-91d2-95f1e4c1b2a9%40postgresfriends.org), [#6](https://www.postgresql.org/message-id/CAMT0RQRjvzGYNgUBzmRaRuX64m-VM2n6Stm3ANmzigoSb_Wf1w%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAMT0RQShho4N09N1BnPRjPts7SGxiM5pH8SK5tGBC%3D4wAoxkDQ%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Non-recursive UNION DISTINCT ON has no use case beyond a workaround

A user who wants the newest row per key across two tables writes DISTINCT ON over a UNION ALL subquery today, and it works. With the patch the project carries new set-operation grammar, Append-flattening guards and pushdown checks for that case. No user story shows the subquery form failing.

In message #4 Vik Fearing asked for a UNION DISTINCT ON use case, and message #5 gave only a recursive shortest-path query. In message #8 the non-recursive form was 1.36x slower than the classic rewrite at 200k groups. Message #10 shows a gain only with an unmerged estimate diff, and a loss again at 500k groups. The discussion paper argues for early pruning but gives no workload where the rewrite is too slow.

**Question:** Which real non-recursive queries need UNION DISTINCT ON, and why is DISTINCT ON over a UNION ALL subquery not enough for them?

<sub>fixable, level 1b, demand; messages [#4](https://www.postgresql.org/message-id/e9b65da3-9326-4951-91d2-95f1e4c1b2a9%40postgresfriends.org), [#5](https://www.postgresql.org/message-id/CAMT0RQRCxNHoOjLUnFsZk8aNQqFDkYs3n2P5HecKYigAGJk0xQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/179226324434.751863.15815974755418841666%40gmail.com), [#10](https://www.postgresql.org/message-id/179302661466.1417973.8633344727673033675%40gmail.com); `demand-1`</sub>

## Questions for the author

1. Has WG 3 seen the UNION DISTINCT ON form, and would the Postgres design still hold if the committee chose USING KEY or different semantics?
2. Which real non-recursive queries need UNION DISTINCT ON, and why is DISTINCT ON over a UNION ALL subquery not enough for them?
3. Can the series be reordered so that 0001 plus the inline ORDER BY patch stand alone, each correct without 0005?
4. Will the docs define the recursive mode as replacement, state the negative-cycle case, and explain why SEARCH and CYCLE are rejected with an inline ORDER BY?
5. Does the result of recursive pruning depend on the order in which rows arrive, and if so, in which cases?

## Touch points

- **ABI / extension API:** The series adds fields to parse, path and plan nodes and a new ReplaceTupleHashEntryIfBetter() in executor.h.
- **SQL-standard conflict or non-standard syntax:** It adds DISTINCT ON (keys ORDER BY ...) and UNION DISTINCT ON, both non-standard, with only a draft discussion paper behind them.
- **planner behaviour change:** It adds HashAggregate and MergeAppend plus Unique paths for DISTINCT ON and changes set-operation grouping and qual pushdown.
- **pg_dump / restore round-trip:** Views can contain both new forms, so dump and restore depend on the new pg_get_viewdef() deparsing in ruleutils.c.
- **backpatchability:** see objection standards-compat-1: New DISTINCT ON spellings ship before the standard committee has reviewed them
- **maintenance burden ("who maintains this in 10 years"):** RecursiveUnion gains a replace-and-requeue mode, and about 30 files change across parser, planner, executor and rewriter.
- **performance on the hot path:** The series changes execGrouping.c and nodeAgg.c, which every hashed aggregate uses; message #8 reports no measurable regression.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Non-recursive UNION DISTINCT ON can be written today as DISTINCT ON over a UNION ALL subquery.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows set-operation planning, hashed aggregation and nodeRecursiveunion.c. Only Vik Fearing, on syntax, and Manu, on testing, have replied.
- Priority: needs design rethink; audience a sizeable minority; review cost high. The inline ORDER BY answers a common pattern and has support in message #4, but the series ties it to UNION DISTINCT ON and the recursive mode. Those two parts carry the open syntax and demand questions, and the series spans about 30 files.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
