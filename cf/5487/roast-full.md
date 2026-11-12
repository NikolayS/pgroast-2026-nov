# Design pre-review: Pull-up subquery if INNER JOIN-ON contains refs to upper-query

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5487](https://commitfest.postgresql.org/patch/5487/) (Needs review); authors: Alena Rybakina (a.rybakina)
- Thread: [Exists pull-up application with JoinExpr](https://www.postgresql.org/message-id/flat/0b1f670d-b39d-4966-bf32-f0d502ebc564%40postgrespro.ru), started 2024-12-24
- Messages read: 30 of 30
- Patch prose read: v5-0001-Enables-pull-up-of-EXISTS-subqueries-tha.patch, 0001-Add-EXISTS-pull-up-if-subquery-join-expressions.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

Level 1b is where a committer would likely stop. already-exists-1: message #1 shows the WHERE form already gets the semi join, and the thread names no workload that emits the ON form and cannot change it. demand-1: 0002 rests on one Cloudberry discussion, and message #30 measures a query going from about 0.2 ms to 36 ms. Both are requests for evidence, and reviewers report 0001 correct. Confidence is low because the verdict rests on missing demand evidence and only 3 of 30 messages are visible.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the nullable-side rules of 0001, the row equivalence of the 0002 rewrite and the handling of security-barrier quals could not be checked. The file list and commit messages for v8 to v11 are not in the material.

## The proposal, restated

**Problem.** The planner turns `exists (select ... from tb, tc where ta.id = tb.id)` into a semi join, but keeps a SubPlan when the same correlation is written as `tb join tc on ta.id = tb.id`. Users who write correlated conditions in ON clauses get a per-row SubPlan and lose join reordering and index use.

**Proposed design.** The 0001 patch takes the jointree out of the sub-select, checks that nothing else refers to the parent, then walks the jointree with a nullable-side flag. Correlated quals in ON clauses of inner joins not under an outer join's nullable side move to WHERE, and the sublink becomes a semi or anti join. The 0002 patch handles EXISTS in an outer join's ON clause referencing both inputs, using a LEFT JOIN against distinct correlation keys plus a CASE check.

**User-visible surface.**
- Planner change: more EXISTS and NOT EXISTS sublinks become semi or anti joins, so plans change.
- v5 touched guc_tables.c and postgresql.conf.sample, so it added a GUC; the thread never names it or says if later versions keep it.

**What the author claims.**
- After the patch, ON-clause correlation gives the same plan as WHERE-clause correlation.
- SQLsmith and a one-hour SQLancer run found no new errors with v10 and v11.
- v8 fixes wrong results with outer joins found in v7 and leaves join.out unchanged.
- 0002 follows a Cloudberry bug discussion about EXISTS in outer join ON clauses.

**Thread state.** Ilia Evdokimov says 0001 looks good after random-query testing found no wrong results; Solaimurugan V also tested earlier versions. Open: Ilia showed a large slowdown from 0002, which picks its fallback before path generation, so no cost comparison with the SubPlan is possible.

## 2. Prior attempts

### Pull up sublink of type 'NOT NOT (expr)' (related)

- **Link:** [https://commitfest.postgresql.org/patch/1831/](https://commitfest.postgresql.org/patch/1831/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com)
- **Date:** decided 2018-11-23; started 2018-10-17
- **Outcome:** Rejected (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane argued that pull_up_sublinks runs before transformations that could create NOT NOT, so it only sees NOT NOT that a user wrote by hand, and no one showed that this happens in real queries. Without such evidence the extra code and planning cycles would be a net loss for everyone else. Running expression simplification before pullup would need a second simplification pass, which he judged not worth the cost. The author agreed and asked for the patch to be rejected.
- **Reason (quote):** "So I'm inclined to reject this patch as probably being a net loss in almost all cases. If you can show any real-world cases where it wins, we could reconsider." ([message](https://www.postgresql.org/message-id/29888.1542319596%40sss.pgh.pa.us))
- **What is different this time:** P3 targeted a hand-written NOT NOT form. 0001 targets correlation written in an inner join ON clause, but the thread likewise shows no real workload that uses the form.

### Pulling up direct-correlated ANY_SUBLINK (related)

- **Link:** [https://commitfest.postgresql.org/patch/3941/](https://commitfest.postgresql.org/patch/3941/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAKU4AWoZksNZ4VR-fLTdwmiR91WU8qViDBNQKNwY=7iyo+uV0w@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAKU4AWoZksNZ4VR-fLTdwmiR91WU8qViDBNQKNwY=7iyo+uV0w@mail.gmail.com)
- **Date:** decided 2023-01-31; started 2022-11-02
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Richard Guo (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane found the patch messy because it duplicated code from elsewhere, and he posted a different approach that drops the contain_vars_of_level restriction and uses LATERAL. Richard Guo showed that without a limit on outer references, LATERAL constraints can leave no legal join order ("failed to build any 3-way joins"). The v2 patch then stopped applying on HEAD, nobody posted a rebased version, and the entry was returned with feedback.
- **Reason (quote):** "This patch seems awfully messy to me. The fact that you're having to duplicate stuff done elsewhere suggests at the least that you've not plugged the code into the best place." ([message](https://www.postgresql.org/message-id/247851.1668293143%40sss.pgh.pa.us))
- **What is different this time:** P1 pulled up correlated ANY sublinks through LATERAL. 0001 moves ON-clause quals to WHERE and builds a plain semi or anti join.


## 3. Objections a reviewer is likely to raise

### 1. Moving the correlated qual to WHERE already gives the semi join

fixable · level 1b, rubric item 3 · desirability, planner behaviour change, doesn't fit what Postgres is about (extension / external tool / already solved) · `already-exists-1`

A user whose ON-correlated EXISTS runs well as a SubPlan upgrades, and 0001 now pulls it up into a slower plan, as in message #12 (about 70 s instead of 68 s). Users who wanted the semi join could already get it by writing the correlation in WHERE.

For inner joins not under an outer join's nullable side, 0001 moves the ON qual to WHERE. A user can make the same rewrite by hand today, and message #1 shows that the WHERE form already gets a Nested Loop Semi Join. The thread shows no field report, ORM output or -general thread where users write or generate the ON form and cannot change it; the only outside motivation, the Cloudberry case in message #29, is for 0002. In [P3], a pull-up of hand-written forms was rejected until real-world cases were shown.

**Precedents.**
- [Pull up sublink of type 'NOT NOT (expr)'](https://commitfest.postgresql.org/patch/1831/) (Rejected, 2018-11-23); thread: [link](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com)

Thread: [#1 (Alena Rybakina, 2024-12-24)](https://www.postgresql.org/message-id/0b1f670d-b39d-4966-bf32-f0d502ebc564%40postgrespro.ru), [#12 (Alena Rybakina, 2025-06-04)](https://www.postgresql.org/message-id/87a6fcbf-929a-408a-9471-28be3080254a%40postgrespro.ru), [#29 (Alena Rybakina, 2026-10-13)](https://www.postgresql.org/message-id/600857c0-8439-4c9c-8d87-6cb46dfc4406%40yandex.ru)

**Question:** Which applications or query generators emit correlated EXISTS with the correlation in an inner join ON clause and cannot be changed to put it in WHERE?

Counter-check: held. Message #1 shows the WHERE form already gets the semi join, and the thread cites no workload or query generator that emits the ON form. The existing code comment quoted in message #19 calls this case "probably not worth the trouble", so [P3] applies.

### 2. 0002 rests on one external bug discussion and slows queries that were fast

fixable · level 1b, rubric item 4 · demand, planner behaviour change, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A user has a LEFT JOIN whose ON clause has an EXISTS that is fast today as an index-only SubPlan. After upgrade, 0002 picks the LEFT JOIN plus HashAggregate fallback before path generation. The query reads and groups the whole inner table, and the user finds out from a slower production query.

The only stated motivation for 0002 is a Cloudberry bug discussion (message #29), which comes from a fork, and no Postgres user report is cited. Message #30 shows a query going from about 0.2 ms to 36 ms because the rewrite is chosen without costing. The cost falls on users whose queries already run well, while the gain is for an unmeasured group. [P3] was rejected when nobody showed real cases where the pull-up wins, and the same question applies here.

**Precedents.**
- [Pull up sublink of type 'NOT NOT (expr)'](https://commitfest.postgresql.org/patch/1831/) (Rejected, 2018-11-23); thread: [link](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com)

Thread: [#29 (Alena Rybakina, 2026-10-13)](https://www.postgresql.org/message-id/600857c0-8439-4c9c-8d87-6cb46dfc4406%40yandex.ru), [#30 (Ilia Evdokimov, 2026-10-19)](https://www.postgresql.org/message-id/CAAgWR6ZPLnEEJw67SkF%2BM8VVn-EK0e%3DbGX5BovQraKhUexS%2BCw%40mail.gmail.com)

**Question:** Which Postgres workloads put a correlated EXISTS in an outer join's ON clause, and how often is the fallback faster there than the SubPlan?

Counter-check: held. Message #29 names only a Cloudberry discussion as the motivation for 0002, and the thread shows no measurement where the fallback wins. Message #30 shows that the cost falls on a query that is fast today.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v5 touches src/include/optimizer/optimizer.h, a header that planner extensions include. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | v5 touches guc_tables.c and postgresql.conf.sample; the thread does not name the GUC or say if v11 keeps it. |
| planner behaviour change | **yes** | More EXISTS and NOT EXISTS sublinks become semi, anti or left joins, so existing plans change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new jointree walker with nullable-side rules in subselect.c must stay correct through later outer-join changes. |
| performance on the hot path | **yes** | 0002 picks its rewrite before path generation; message #30 shows a query about 175 times slower. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A user can already get the semi join by writing the correlation in WHERE, as message #1 shows. |

## 5. Questions the author must answer before anyone reviews code

1. Which applications or query generators emit a correlated EXISTS with the correlation in an inner join ON clause and cannot be changed to put it in WHERE?
2. Which Postgres workloads put a correlated EXISTS in an outer join's ON clause, and how often is the 0002 fallback faster there than the SubPlan?
3. How will 0002 keep the SubPlan available when it is cheaper, so the query in message #30 does not get slower after upgrade?
4. What argument, beyond SQLsmith and SQLancer runs, shows that the distinct-key LEFT JOIN with the CASE check returns the same rows as the SubPlan?
5. Does v11 still add the GUC that v5 added, and if so what are its name and default?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows pull_up_sublinks and the outer-join nullable-side rules in the planner. None has commented in the visible thread or is listed on the CommitFest entry.
- **Liability:** The owner signs for the correctness of EXISTS results under every outer-join shape, with no end date, and for plan regressions in existing queries after upgrade.
- **Author follow-up:** The author has posted eleven versions since 2024-12 and still moves the entry between CommitFests, so follow-up looks likely.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: For 0001: real queries, ORM output or user reports that put the correlation in an inner join ON clause and cannot be rewritten, with measured gains. For 0002, demand evidence is not enough alone; it also needs a way to keep the SubPlan when it is cheaper.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Reviewers report 0001 correct after random-query testing, but no committer has engaged in 10 CommitFests and the thread shows no user demand. The v5 patch was about 54 KiB in subselect.c with subtle nullable-side rules, and 0002 still has an open design question from message #30.

## Appendix

### Held up in the counter-check, not listed above

- `prior-attempts-1` (fixable, level 2) Patch 0002 rewrites to a de-duplicated LEFT JOIN without costing or proof. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-1` (fixable, level 2) 0002 replaces working SubPlan plans with an uncosted fallback. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-1` 0002 picks the LEFT JOIN fallback before costing, with no way back. Refuted (not specific): Duplicate of standards-compat-1 for the uncosted fallback and of demand-1 for the missing user reports. A new version of 0002 could solve this, and 0001 does not depend on it, so it would not be fatal.
- `maintenance-fit-2` No committer who knows sublink pull-up has engaged in two years. Refuted (no failure scenario): The scenario is a possible future bug from an unnamed later change, and the v7 wrong results were fixed in v8 and then tested in message #26. That no committer has commented is true, but it belongs in the committability field.
- `maintenance-fit-3` Bundling 0002 with 0001 ties a reviewed change to an unresolved one. Refuted (already handled): 0001 was posted alone as v8 to v10 and reviewed in that form, and message #30 approves v11-0001 separately. Message #29 places the shared preparation in the second patch, so the series is infrastructure first and separately applicable.
- `demand-2` No field evidence for 0001, and the WHERE rewrite already works. Refuted (not specific): Duplicate of already-exists-1, which makes the same demand and manual-rewrite argument more precisely. The wrong results were fixed in v8, so "eleven versions" overstates that point.

### Review angles

- prior-attempts: 1 objection(s). No precedent matches 0001. It applies the existing non-costed WHERE-clause semi join conversion to ON clauses, and reviewers in messages #26 and #30 report it correct. The one objection concerns 0002 only.
- standards-compat: 1 objection(s). 0001 makes ON-clause correlation behave like WHERE-clause correlation, which the planner already pulls up without costing. The slower plan in message #12 follows that existing practice, so I raise no objection to 0001. I could not check whether the v5 GUC still exists in v11.
- replication-ondisk: 0 objection(s). The proposal changes only planner rewriting of EXISTS sublinks. It adds no on-disk format, WAL record, catalog column, wire protocol message or replication behaviour. A possible GUC from v5 touches no persistent state. Standbys and subscribers do not see plan choices. Plans do not cross nodes, so no extension ABI concern is visible from the material.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. Both patches change only how the planner turns EXISTS sublinks the user already wrote into semi, anti or left joins, so no new privilege, role, function or visible data is involved. Qual ordering against RLS and security barrier quals uses the same security_level handling as today's WHERE-correlated pull-up. Without the code, I could not check that moved ON-clause quals keep that handling.
- maintenance-fit: 3 objection(s). 0001 alone copies the existing uncosted EXISTS pull-up for WHERE-clause correlation. So the regression in message #12 is existing behavior and is not raised here. The v5 GUC could not be checked.
- already-exists: 1 objection(s). 0002 has no simple user rewrite in the material, so this angle raises nothing against it.
- demand: 2 objection(s). Two objections are raised. Both ask for evidence of demand and are not rejections of the idea.

### Precedents retrieved

- P1: [Pulling up direct-correlated ANY_SUBLINK](https://commitfest.postgresql.org/patch/3941/) (cf, Returned with feedback, 2023-01-31) - used in the review; reason summarised from its thread
- P2: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - judged not relevant; reason summarised from its thread
- P3: [Pull up sublink of type 'NOT NOT (expr)'](https://commitfest.postgresql.org/patch/1831/) (cf, Rejected, 2018-11-23) - used in the review; reason summarised from its thread
- P4: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P5: [NOT IN to ANTI JOIN transformation](https://commitfest.postgresql.org/patch/2023/) (cf, Returned with feedback, 2020-04-08) - used in the review; reason summarised from its thread
- P6: [Convert NOT IN to anti-joins when possible](https://commitfest.postgresql.org/patch/2020/) (cf, Returned with feedback, 2019-11-28) - used in the review; reason summarised from its thread
- P7: [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (cf, Returned with feedback, 2018-10-02) - used in the review; reason summarised from its thread
- P8: [CREATE ROLE IF NOT EXISTS](https://commitfest.postgresql.org/patch/3370/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P9: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P10: [Join Correlation Name](https://commitfest.postgresql.org/patch/2322/) (cf, Withdrawn, 2019-11-01) - judged not relevant
- P11: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P12: [INNER JOIN removals](https://commitfest.postgresql.org/patch/27/) (cf, Rejected, 2015-02-14) - judged not relevant
- P13: [IF NOT EXISTS for PREPARE](https://commitfest.postgresql.org/patch/585/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P14: [Select Reference Page - Make Join Syntax More Prominent](https://commitfest.postgresql.org/patch/3726/) (cf, Returned with feedback, 2022-09-10) - judged not relevant
- P15: [Revert pg_bind_textdomain_codeset to a existant-but-empty function when ENABLE_NLS is not defined, for better compatibility of the backend with modules...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e43fd89762a5a894ea04f4fe74e05f95a1488664) (revert, Reverted, 2009-03-09) - judged not relevant
- P16: [Detect the null type correctly in setop case by peeking the later exprs.](https://commitfest.postgresql.org/patch/2653/) (cf, Withdrawn, 2020-07-23) - judged not relevant
- P17: [Revert "Fix an O(N^2) problem in foreign key references".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3d9e8db9e56beb0501d168779ae9644587924061) (revert, Reverted, 2015-09-15) - judged not relevant
- P18: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P19: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P20: [Revert: Remove useless self-joins](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d286d83c0eed695910cb20d970ea9bea2e5001) (revert, Reverted, 2024-05-06) - judged not relevant
- P21: [pg_convert improvement](https://commitfest.postgresql.org/patch/4673/) (cf, Withdrawn, 2023-11-28) - judged not relevant
- P22: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P23: [Revert "Add key management system" (978f869b99) & later commits](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3187ef7c46c5b884267a88f2d6119c9a05f1bbba) (revert, Reverted, 2020-12-28) - judged not relevant
- P24: [rename column if exists](https://commitfest.postgresql.org/patch/3045/) (cf, Rejected, 2022-01-17) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 170 s; cost reported by the CLI: $3.46
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 90 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:32:11.450Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
