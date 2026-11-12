# Design pre-review: FROM clause before SELECT

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7122](https://commitfest.postgresql.org/patch/7122/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [FROM clause before SELECT](https://www.postgresql.org/message-id/flat/8B32D9DE-6152-485F-AB53-9405A04C757A%40yandex-team.ru), started 2026-07-27
- Messages read: 4 of 4
- Patch prose read: v2-0001-Allow-a-query-to-be-written-with-its-select-list-.patch, 0001-Allow-the-FROM-clause-to-be-written-before-the-SELEC.patch, 0002-psql-complete-column-names-in-FROM-first-queries.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Level 1b fails on three fixable objections. The main argument is psql completion, but the proposal does not say why psql cannot use the text after the cursor (already-exists-1). The only outside demand is for the v1 order, which v2 rejects (demand-1). Message #4 says completion centers on a single relation (prior-attempts-1). The goal may be fine, and the thread has one supporter, so this is a discussion to finish before code review.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the claim of no new grammar conflicts and the effect on parser size could not be checked. The material also does not show what the completion tests cover under the v2 order.

## The proposal, restated

**Problem.** Postgres requires a query to start with SELECT, so psql cannot complete column names in the select list: the completer sees only text left of the cursor, and the tables come later. The author frames this as help for interactive users, and admits the real motivation is partly "just for fun".

**Proposed design.** v2 lets a query start with FROM, followed by WHERE, GROUP BY, HAVING and WINDOW in their usual order, then the select list, then ORDER BY, LIMIT and locking clauses. An omitted SELECT means SELECT *. The grammar builds the same SelectStmt as the standard form, so nothing below the parser changes; v1 instead put the select list right after FROM.

**User-visible surface.**
- New query syntax: FROM from_item [other clauses] SELECT select_list, accepted wherever a SELECT is, including sub-selects, CTEs, set operations and INSERT ... SELECT.
- Bare FROM with SELECT omitted means SELECT *, for example FROM films WHERE ...
- PL/pgSQL recognizes FROM as the start of a SQL statement.
- psql tab completion offers FROM as a command start and completes columns in the select list and in GROUP BY.
- New sections in the SELECT reference page, including a Compatibility note calling the form a PostgreSQL extension.

**What the author claims.**
- The new productions add no grammar conflicts and change nothing below the parser.
- Stored queries and pg_get_viewdef output show the standard form.
- FROM-first order lets psql complete select-list columns, which is impossible when SELECT comes first.
- Select-list-last is easier to teach than allowing the select list in several positions.
- The standard will never move SELECT, so Postgres is free to define this extension.

**Thread state.** Vik Fearing supports the idea and prefers SELECT after HAVING, which v2 adopts. Denis Smirnov prefers v1 for DuckDB and ClickHouse compatibility and output-alias completion in GROUP BY, and asks for JOIN completion tests; Peter Eisentraut is the listed committer but has not posted.

## 2. Prior attempts

### Tab completion for SELECT in psql (related)

- **Link:** [https://commitfest.postgresql.org/patch/1376/](https://commitfest.postgresql.org/patch/1376/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAMyN-kB_xrU4iYdcF1j=tijgO1DSyjtb3j96O4UEj91XZrZcMg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAMyN-kB_xrU4iYdcF1j=tijgO1DSyjtb3j96O4UEj91XZrZcMg@mail.gmail.com)
- **Date:** decided 2018-10-01; started 2017-11-06
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Heikki Linnakangas, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Heikki Linnakangas found the completions both too broad and too narrow: system catalog columns flood short prefixes, the FROM clause is ignored, and system columns appear inconsistently. He held that the completion must be smarter to be acceptable. He also reported that the query on pg_attribute took about 1 second on a database with a 1000-partition table. An earlier review by Peter Eisentraut found broken older-version queries and an unsettled function filter. The rebased patch of 2018-09-22 got no visible reply before the entry was returned.
- **Reason (quote):** "I think the completion needs to be smarter to be acceptable." ([message](https://www.postgresql.org/message-id/03404486-d6bf-5cb3-c937-c4590a542402%40iki.fi))
- **What is different this time:** P3 completed select-list columns without knowing the tables. This proposal changes the grammar so the tables are named first and completion can use them.


## 3. Objections a reviewer is likely to raise

### 1. A psql completion gap is being fixed with server grammar

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), SQL-standard conflict or non-standard syntax, psql UX conventions, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An application developer pastes a FROM-first query from a psql session into application code. A SQL linter or proxy that embeds a Postgres-compatible parser rejects it, and the developer learns this only at deploy time. The only stated gain was select-list completion inside psql.

Message #1 and the 0002 commit message present completion as the practical argument. They say the completer sees only text left of the cursor. That is a design choice in psql's tab-complete code: readline also exposes the rest of the line, and a client such as pgcli reads the whole buffer and offers select-list columns for SELECT ... FROM t. A committer would likely ask why a client-side limit justifies a permanent server grammar extension.

Thread: [#1 (Andrey Borodin, 2026-07-27)](https://www.postgresql.org/message-id/8B32D9DE-6152-485F-AB53-9405A04C757A%40yandex-team.ru), [#3 (Andrey Borodin, 2026-07-28)](https://www.postgresql.org/message-id/9B4457AA-6BFD-45F5-9F24-88AA9A607EDE%40yandex-team.ru)

**Question:** Why can psql not read the text after the cursor, for example a typed "SELECT  FROM t", to complete select-list columns without any grammar change?

Counter-check: held. Readline gives psql the whole line buffer, so the claim that a client "has no way" to complete the select list is too strong. The proposal does not say why a client-side change would not do.

### 2. Demand shown is for v1 order, but v2 ships an order nobody uses

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), SQL-standard conflict or non-standard syntax · `demand-1`

A DuckDB or ClickHouse user pastes `FROM films SELECT title WHERE len > '02:00'` into psql and gets a syntax error. The only outside users who already write FROM-first queries cannot reuse them. The v2 form is then used only by people who learn it fresh from the Postgres docs.

The only demand outside the thread is that two other systems ship the v1 placement (messages #1 and #3). Message #3 admits that "nobody ... ships the placement in v2-patch grammar", and message #4 asks for v1 for that reason. The rest of the demand is the author's teaching sessions and the stated motivation "just for fun" (message #1). No -general or -bugs reports and no user requests appear in the material.

Thread: [#1 (Andrey Borodin, 2026-07-27)](https://www.postgresql.org/message-id/8B32D9DE-6152-485F-AB53-9405A04C757A%40yandex-team.ru), [#3 (Andrey Borodin, 2026-07-28)](https://www.postgresql.org/message-id/9B4457AA-6BFD-45F5-9F24-88AA9A607EDE%40yandex-team.ru), [#4 (Denis Smirnov, 2026-07-30)](https://www.postgresql.org/message-id/8092B28C-07B7-419F-9769-E89850D2D843%40gmail.com)

**Question:** Which users outside this thread have asked for FROM-first queries in Postgres, and which placement do they actually write today?

Counter-check: held. Two other systems shipping FROM-first counts as demand evidence, but only for the v1 order. For v2 the material shows the author's teaching sessions and one supportive reply, with no user requests.

### 3. Completion argument may repeat the quality gap that stalled earlier select-list completion

fixable · level 1b, rubric item 2 · psql UX conventions, SQL-standard conflict or non-standard syntax · `prior-attempts-1`

A user writes `from orders o join customers c using (id) select ` and presses tab. psql offers columns of one table only, or bare names that are ambiguous across both. The permanent grammar extension is then in the release, but the completion that justified it falls short in the common multi-table case.

Message #1 presents 0002 as "the actual pro-argument", and the commit message calls it "the practical argument". [P3] stalled because reviewers found select-list completion too broad and too narrow, and its key sentence asked for smarter completion. Message #4 reports that the current rules are "mostly centered on a single relation" and asks for tests with joins, aliases and qualified names. The material does not show that 0002 was updated for v2 placement, so the main benefit is not yet shown for the queries users write.

**Precedents.**
- [Tab completion for SELECT in psql](https://commitfest.postgresql.org/patch/1376/) (Returned with feedback, 2018-10-01); thread: [link](https://www.postgresql.org/message-id/flat/CAMyN-kB_xrU4iYdcF1j=tijgO1DSyjtb3j96O4UEj91XZrZcMg@mail.gmail.com)

Thread: [#1 (Andrey Borodin, 2026-07-27)](https://www.postgresql.org/message-id/8B32D9DE-6152-485F-AB53-9405A04C757A%40yandex-team.ru), [#3 (Andrey Borodin, 2026-07-28)](https://www.postgresql.org/message-id/9B4457AA-6BFD-45F5-9F24-88AA9A607EDE%40yandex-team.ru), [#4 (Denis Smirnov, 2026-07-30)](https://www.postgresql.org/message-id/8092B28C-07B7-419F-9769-E89850D2D843%40gmail.com)

**Question:** How does the v2 completion behave for joins, table aliases and qualified columns, and do the tests cover those cases?

Counter-check: held. Message #4 says the rules center on a single relation, and nothing in the thread answers it. Two supports are weak: [P3] ignored FROM, which this patch knows, and v2-0001 does touch tab-complete.in.c and its test.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds a non-standard query form that starts with FROM, with the select list last or omitted. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | The patch changes tab-complete.in.c to offer FROM as a command start and to complete select-list and GROUP BY columns. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | gram.y gets parallel productions that each future SELECT clause must also reach, and pl_gram.y learns FROM as a statement start. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The stated gain is psql completion, which a client-side change might deliver without new server grammar. |

## 5. Questions the author must answer before anyone reviews code

1. Why can psql not read the text after the cursor, for example a typed "SELECT FROM t", to complete select-list columns without a grammar change?
2. Which users outside this thread have asked for FROM-first queries in Postgres, and which clause order do they write today?
3. How does the v2 completion behave for joins, table aliases and qualified columns, and do the tests cover those cases?
4. Why should Postgres define the v2 select-list-last order, which no other system accepts, when DuckDB and ClickHouse users already write the v1 order?
5. What does the bare FROM form give users beyond TABLE and FROM ... SELECT *, given that a truncated statement would then run as SELECT *?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut added himself as committer in the CommitFest entry on 2026-11-02 but has not posted in the visible thread. The owner must know gram.y and the SQL standard.
- **Liability:** A non-standard query spelling that cannot be removed once users write it, plus parallel grammar productions and psql completion rules to keep in step with every later SELECT change.
- **Author follow-up:** The author posted v2 one day after the first review and moved the entry to the next CommitFest. Message #1 also says he will drop the patch if consensus is against non-standard grammar.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The shown benefit is select-list completion for psql users, and message #4 says it centers on a single relation. The patch is small, but it adds permanent syntax to gram.y, and the thread has not chosen between the v1 and v2 order.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) v2 clause order matches no other system and fixes a third dialect. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-3` (fixable, level 2) Bare FROM adds a second SELECT * shorthand next to TABLE. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) A second query grammar that every future SELECT clause must track. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-2` Permanent SQL syntax added to work around a psql completer limit. Refuted (not specific): Duplicate of already-exists-1, which makes the same point about a client-side limit with a more concrete question. Its completion part repeats prior-attempts-1.
- `maintenance-fit-2` v2 placement matches no other system and cannot change once shipped. Refuted (not specific): Duplicate of standards-compat-1. Its added scenario does not hold, because adding v1 placement later would not break stored v2 queries, and [P8] concerned a keyword the standard already listed.
- `maintenance-fit-3` Small completion gain weighed against a permanent syntax extension. Refuted (not specific): Duplicate of demand-1 for the missing demand and of prior-attempts-1 for join completion. Its accumulation part repeats demand-3.
- `already-exists-2` Bare FROM duplicates TABLE and SELECT * with little saved. Refuted (not specific): Duplicate of standards-compat-3, which adds a concrete scenario. [P8] must not be used as support, since its recorded reason was a keyword conflict with the standard.
- `demand-2` Gain is one psql clause, cost falls on every SQL parser. Refuted (not specific): Every grammar addition, including standard ones such as MERGE, requires external parsers to catch up. Nothing ties this cost to this proposal, and the project does not carry it.
- `demand-3` The case for this applies equally to the friendly-SQL list. Refuted (already handled): Message #1 gives the criterion: the surface is "tied to the completion argument", which ORDER BY ALL, EXCLUDE and trailing commas lack. The message leaves those out as separate discussions.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes FROM-first or select-list-last syntax. [P8]'s standard-conflict reason is weak here because FROM is already reserved, so I raise no objection on it.
- standards-compat: 3 objection(s). Not applicable. pg_dump and pg_get_viewdef output stays in the standard form, so this review raises no round-trip objection.
- replication-ondisk: 0 objection(s). The proposal adds no persistent or cross-node state. The grammar builds the same SelectStmt, so catalogs, WAL, on-disk format, the wire protocol and replication are untouched, and stored views and BEGIN ATOMIC bodies deparse to the standard form. Text-stored function bodies keep the new spelling and will not load on older servers, but the same holds for any new syntax.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The new spelling builds the same SelectStmt as the standard form, so permission checks, RLS, column privileges and security barrier views apply unchanged. psql completion reads column names from catalogs that are already readable by every user.
- maintenance-fit: 3 objection(s). Objections are listed above. There is no hot-path cost: the change is limited to the parser. The listed committer has not posted yet.
- already-exists: 2 objection(s). Two objections raised from the already-exists angle.
- demand: 3 objection(s). Demand is thin but not shown to be absent, so all objections are "not now, show demand" and fixable.

### Precedents retrieved

- P1: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P2: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant; reason summarised from its thread
- P3: [Tab completion for SELECT in psql](https://commitfest.postgresql.org/patch/1376/) (cf, Returned with feedback, 2018-10-01) - used in the review; reason summarised from its thread
- P4: [INSERT .. ON CONFLICT DO SELECT](https://commitfest.postgresql.org/patch/1241/) (cf, Returned with feedback, 2018-01-22) - judged not relevant
- P5: [INSERT ... ON CONFLICT DO SELECT \[FOR ...\]](https://commitfest.postgresql.org/patch/5413/) (cf, Returned with feedback, 2025-08-05) - judged not relevant; reason summarised from its thread
- P6: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - judged not relevant; reason summarised from its thread
- P7: [Add SQL syntax check](https://commitfest.postgresql.org/patch/4704/) (cf, Rejected, 2024-05-15) - judged not relevant; reason summarised from its thread
- P8: [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (cf, Rejected, 2018-07-19) - used in the review; reason summarised from its thread
- P9: [Boolean partition syntax](https://commitfest.postgresql.org/patch/1410/) (cf, Returned with feedback, 2018-03-21) - judged not relevant; reason summarised from its thread
- P10: [Ltree syntax improvement](https://commitfest.postgresql.org/patch/1977/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P11: [Implement INSERT SET syntax](https://commitfest.postgresql.org/patch/2218/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P12: [SET syntax for INSERT](https://commitfest.postgresql.org/patch/770/) (cf, Returned with feedback, 2016-09-05) - judged not relevant
- P13: [Relax requirement for INTO with SELECT in pl/pgsql](https://commitfest.postgresql.org/patch/586/) (cf, Returned with feedback, 2016-09-22) - judged not relevant
- P14: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P15: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant
- P16: [psql: Add tab-complete for optional view parameters](https://commitfest.postgresql.org/patch/4053/) (cf, Returned with feedback, 2023-07-06) - judged not relevant
- P17: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P18: [Add last commit LSN to pg_last_committed_xact()](https://commitfest.postgresql.org/patch/3515/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P19: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant
- P20: [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (cf, Returned with feedback, 2021-11-26) - judged not relevant
- P21: [remove C++ keywords](https://commitfest.postgresql.org/patch/1075/) (cf, Returned with feedback, 2017-04-06) - judged not relevant
- P22: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - judged not relevant
- P23: [Reorder tablespaces for streaming basebackups](https://commitfest.postgresql.org/patch/1035/) (cf, Rejected, 2017-03-31) - judged not relevant
- P24: [Add initdb option to initialize cluster with non-standard xid/mxid/mxoff.](https://commitfest.postgresql.org/patch/3634/) (cf, Rejected, 2023-03-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 154 s; cost reported by the CLI: $1.87
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 113 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:22:03.162Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
