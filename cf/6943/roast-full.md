# Design pre-review: ON EMPTY clause for aggregate and window functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6943](https://commitfest.postgresql.org/patch/6943/) (Needs review); authors: Jeevan Chalke (jeevan.chalke)
- Thread: [ON EMPTY clause for aggregate and window functions](https://www.postgresql.org/message-id/flat/CAM2%2B6%3DVS%3DfSKxfimW6Th9iu_xjbxOEAKg4eYwaa%3DSMg3X8pHaQ%40mail.gmail.com), started 2026-06-26
- Messages read: 14 of 14
- Patch prose read: nocfbot-wip-Minimal-ON-EMPTY-sum-product-only-rewritten-to.patch, v1-0001-Add-support-for-ON-EMPTY-clause-in-aggregate-and-.patch, v1-0002-jit-Support-EEOP_AGG_INPUT_RECEIVED-in-the-LLVM-e.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The proposal fails at level 1b. In message #11 the author accepts that v3 never differs from COALESCE, and message #7 covers the remaining case with a count()-based CASE (demand-1). The costs named in message #9 fall on everyone (maintenance-fit-1), the migration need names no database (demand-2), and the standard edition is not published (standards-compat-2). The list must agree that the clause is wanted, and in which form, before code review makes sense.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and the v3 file list is not shown. Whether postgres_fdw and other code that rebuilds or deparses Aggref keep the ON EMPTY field could not be checked.

## The proposal, restated

**Problem.** An aggregate over zero rows returns NULL, and users must wrap it in COALESCE to get a different value. The author cites an upcoming SQL standard revision that defines ON EMPTY for sum() and product(), and Vik Fearing cites migration from other databases that support it.

**Proposed design.** v1 tracked per group whether any row reached the aggregate and returned the given value only on truly empty input, which blocked partial aggregation. v2 and v3 define agg(args, default ON EMPTY) as exactly COALESCE(agg(args), default) for plain and window aggregates. A later WIP patch accepts ON EMPTY only on sum() and product() and rewrites it to COALESCE at parse time.

**User-visible surface.**
- SQL syntax agg_function(args, default_value ON EMPTY), also with ALL, DISTINCT and before WITHIN GROUP.
- ON EMPTY on aggregates used as window functions, returning the value for an empty frame (rejected in the WIP patch).
- New fields in Aggref, WindowFunc and FuncCall stored in pg_rewrite, needing a catversion bump (v1 to v3).
- View and rule deparse shows COALESCE in place of ON EMPTY in the WIP patch.

**What the author claims.**
- ON EMPTY is expected in an upcoming SQL standard revision, defined as COALESCE over the result for SUM and PRODUCT.
- v1 behaved differently for plain and windowed sum() over all-NULL input, because their transition functions differ in strictness.
- The author found no case where the general v3 design differs from COALESCE, and accepts it adds little value.
- The WIP patch is under 150 lines with tests, with no executor or catalog changes.

**Thread state.** Robert Haas and Tom Lane see little value over COALESCE and object to the sum()/product()-only version, and Tom calls the parse-time rewrite an ugly hack. Vik Fearing wants a generic version or none, and no reply came after 2026-09-28.

## 2. Prior attempts

### MAP syntax for arrays (related)

- **Link:** [https://commitfest.postgresql.org/patch/1632/](https://commitfest.postgresql.org/patch/1632/)
- **Thread:** [https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru)
- **Date:** decided 2018-07-19; started 2018-05-04
- **Outcome:** Rejected (cf)
- **Who objected:** Tom Lane, Andreas Karlsson, Ashutosh Bapat (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers objected to adding proprietary syntax that could conflict with future SQL standard use of the MAP keyword, which is already a non-reserved keyword in SQL:2016. They also judged the syntax not much better than the existing sub-select with unnest and array_agg, which is more flexible (for example, it allows a WHERE filter), and suggested optimizing the unnest/array_agg combination in the planner instead of adding new syntax. Heikki Linnakangas summarized the consensus as not wanting the feature and marked the entry Rejected.
- **Reason (quote):** "Yeah, it's a handy syntax, but it's not that much better than using a subselect or a writing a user-defined function. And there's risk of future conflicts with the SQL standard." ([message](https://www.postgresql.org/message-id/14ac5bbe-e9fb-2ce4-2d6d-698f8da29449%40iki.fi))
- **What is different this time:** P9 added syntax outside the standard, while this clause comes from an expected standard edition. Both offer a new spelling for a result that an existing query form already gives.


## 3. Objections a reviewer is likely to raise

### 1. No user need shown beyond what COALESCE already gives

fixable · level 1b, rubric item 2 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A user writes sum(x, 0 ON EMPTY) and gets the same result as coalesce(sum(x), 0), with no other effect. Postgres still keeps a second spelling in gram.y, ruleutils and the docs. Future aggregate work has to keep both spellings in step.

In message #11 the author says he found no case where v3 differs from COALESCE and will no longer argue that it adds value. Tom in message #7 shows that the one case COALESCE cannot cover, an aggregate that returns NULL for non-empty input, is handled by CASE WHEN count(x) > 0. The thread reports no user who asked for this syntax. [P9] was rejected because its syntax was not much better than an existing query form.

**Precedents.**
- [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (Rejected, 2018-07-19); thread: [link](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru)

Thread: [#7 (Tom Lane, 2026-09-17)](https://www.postgresql.org/message-id/636820.1789651904%40sss.pgh.pa.us), [#11 (Jeevan Chalke, 2026-09-24)](https://www.postgresql.org/message-id/CAM2%2B6%3DVaUg4y5H2cpyKvuVgemL7gqdVYtv7hr1nhzJXMzX5HRQ%40mail.gmail.com)

**Question:** Which user, query or field report needs ON EMPTY where coalesce(agg(...), value) or a count()-based CASE is not enough?

Counter-check: held. The author accepts in message #11 that v3 never differs from COALESCE, and message #7 covers the remaining case with a count()-based CASE. The only counterweights are the unpublished standard and an unnamed migration need.

### 2. General ON EMPTY adds grammar and node upkeep for COALESCE semantics

fixable (first rated fatal; changed in the counter-check) · level 1b, rubric item 4 · maintenance burden ("who maintains this in 10 years"), performance on the hot path, catalog changes, SQL-standard conflict or non-standard syntax · `maintenance-fit-1`

Every user pays for the larger Bison grammar on each parse, and committers maintain new Aggref and WindowFunc fields, deparse code and a catversion bump. All of this produces results that users already get from COALESCE(agg(x), v).

Message #11 says v3 gives the same answer as COALESCE in every case the author found. Message #9 names the costs: development, maintenance and some parse-speed penalty, for a small benefit. Message #7 offers a count()-based CASE for the rare aggregate that returns NULL on non-empty input. The cost falls on everyone, and the gain is a second spelling of COALESCE.

**Precedents.**
- [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (Rejected, 2018-07-19); thread: [link](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru)

Thread: [#7 (Tom Lane, 2026-09-17)](https://www.postgresql.org/message-id/636820.1789651904%40sss.pgh.pa.us), [#9 (Tom Lane, 2026-09-17)](https://www.postgresql.org/message-id/645015.1789660502%40sss.pgh.pa.us), [#11 (Jeevan Chalke, 2026-09-24)](https://www.postgresql.org/message-id/CAM2%2B6%3DVaUg4y5H2cpyKvuVgemL7gqdVYtv7hr1nhzJXMzX5HRQ%40mail.gmail.com)

**Question:** Which real migration or user case needs the ON EMPTY spelling instead of COALESCE, and how often does it occur?

Counter-check: held. Message #9 names grammar and maintenance costs that fall on everyone, and message #11 accepts that v3 adds no value over COALESCE. Tom calls the costs small and Robert expects the standard to outweigh them, so publication could change the balance.

### 3. Migration demand is asserted, with no source system or users named

fixable · level 1b, rubric item 3 · demand, SQL-standard conflict or non-standard syntax, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-2`

Postgres adds the grammar rule to help people port queries from other databases. If none of the usual source systems uses this spelling, nobody ports such a query. Everyone still pays the extra grammar and parsing cost Tom describes in message #9.

The only demand in the thread is in message #8, where Vik wants the syntax to help conversion from databases that have it. No database is named, and no migration case is shown. The standard text is quoted second-hand, and the edition is not yet published. Vik says the syntax may still change there. Message #9 says the cost falls on every parse, while the gain goes only to those who write the clause.

Thread: [#8 (Vik Fearing, 2026-09-17)](https://www.postgresql.org/message-id/e9454ec0-6b35-49d6-aac3-7c5f1deeb309%40postgresfriends.org), [#9 (Tom Lane, 2026-09-17)](https://www.postgresql.org/message-id/645015.1789660502%40sss.pgh.pa.us)

**Question:** Which shipping database systems accept agg(x, v ON EMPTY), and is there a migration report where its absence blocked or complicated a port to Postgres?

Counter-check: held. Message #8 is the only statement of demand, and it names no database and no migration case. Naming the systems that accept this syntax would answer it.

### 4. Adopts draft syntax before the standard edition is published

fixable · level 1b, rubric item 6 · SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

Postgres 20 ships agg(x, d ON EMPTY). The published edition changes the spelling, for example to match JSON_VALUE's DEFAULT expr ON EMPTY. Postgres then carries two spellings, or users who adopted the early form find that their SQL does not port.

Message #8 says the next edition is not out and invites a better proposal for the syntax. Tom (message #7) and Robert (message #6) find the current spelling confusing because the substitute value looks like an aggregate argument. The same clause in SQL/JSON, which Postgres already implements, requires DEFAULT before the expression. COALESCE already gives the same result, so the risk of a future conflict is weighed against little gain, as in [P9].

**Precedents.**
- [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (Rejected, 2018-07-19); thread: [link](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru)

Thread: [#6 (Robert Haas, 2026-09-17)](https://www.postgresql.org/message-id/CA%2BTgmoYfZW9y_T8A1nykRUcvJ2FQtMpBjgkq%3D%2BTCdq0-odF4%3Dw%40mail.gmail.com), [#7 (Tom Lane, 2026-09-17)](https://www.postgresql.org/message-id/636820.1789651904%40sss.pgh.pa.us), [#8 (Vik Fearing, 2026-09-17)](https://www.postgresql.org/message-id/e9454ec0-6b35-49d6-aac3-7c5f1deeb309%40postgresfriends.org), [#9 (Tom Lane, 2026-09-17)](https://www.postgresql.org/message-id/645015.1789660502%40sss.pgh.pa.us)

**Question:** Is the ON EMPTY spelling for aggregates final in the standard draft, and why not wait for the published edition before taking it into the grammar?

Counter-check: held. Message #8 confirms the edition is not published and that the syntax could still change. Postgres already implements DEFAULT expr ON EMPTY for SQL/JSON, so a different final spelling is a concrete possibility.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | v1 to v3 add fields to Aggref and WindowFunc that are stored in pg_rewrite, which needs a catversion bump. |
| ABI / extension API | **yes** | New fields in Aggref, WindowFunc and FuncCall affect extensions that walk, copy or deparse these nodes. |
| SQL-standard conflict or non-standard syntax | **yes** | New agg(args, value ON EMPTY) syntax taken from an unpublished standard edition, and extended beyond sum() and product(). |
| new GUC | no |  |
| planner behaviour change | **yes** | v1 marks ON EMPTY aggregates non-partial, and message #10 excludes them from the MIN/MAX index path. |
| pg_dump / restore round-trip | **yes** | The WIP patch deparses views as COALESCE, while v1 to v3 add ON EMPTY deparse in ruleutils.c. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second spelling of COALESCE must be kept in gram.y, node fields, ruleutils.c and every code path that rebuilds an Aggref. |
| performance on the hot path | **yes** | Message #9 names a parse-speed cost from the larger grammar, and v1 adds an executor expression step with JIT support. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | COALESCE(agg(x), v) and a count()-based CASE already give every result the proposal offers. |

## 5. Questions the author must answer before anyone reviews code

1. Which user query, field report or migration case needs ON EMPTY where coalesce(agg(...), value) or a count()-based CASE is not enough?
2. Which shipping database systems accept agg(x, v ON EMPTY), and is there a report where its absence complicated a port to Postgres?
3. Is the ON EMPTY spelling for aggregates final in the standard draft, and why not wait for the published edition before adding it to the grammar?
4. Which version is proposed for commit, the general v3 or the WIP patch limited to sum() and product()?
5. For aggregates other than sum() and product(), why would a later standard edition define ON EMPTY with COALESCE semantics and not the v1 empty-input rule?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. A committer is registered on the CommitFest entry but has not answered the open objections. Tom Lane and Robert Haas take part in the thread, and both object.
- **Liability:** The owner signs for new aggregate grammar and, in v3, new Aggref and WindowFunc fields that every later Aggref rewrite or deparse path must handle. If the published standard differs, the owner also keeps a deviating spelling in stored views.
- **Author follow-up:** The author posted v2, v3, a fix in message #10 and the WIP patch in message #11. The thread has no reply after 2026-09-28.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A published standard edition that fixes this spelling and its semantics, or named database systems that accept agg(x, v ON EMPTY) together with a migration report where its absence was a problem. A user case that COALESCE and a count()-based CASE cannot express would also change the answer.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The thread names no user or source database that needs the clause, and the standard edition it follows is not published. The general version touches grammar, node fields, planner, executor and deparse for results COALESCE already gives, so waiting for the published text costs little.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) General ON EMPTY uses COALESCE semantics that the name does not describe. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-3` (fixable, level 2) Parse-time rewrite makes views dump as COALESCE. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) New Aggref field can be silently dropped by aggregate pushdown in postgres_fdw. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) Subset version hardcodes two aggregates and exposes the rewrite in dumps. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Syntax equivalent to COALESCE, a reason an earlier syntax proposal died. Refuted (not specific): Duplicate of demand-1, which makes the same point without relying mainly on [P9]. [P9] was proprietary syntax at risk of conflict with the standard, while this syntax comes from a standard draft.
- `prior-attempts-2` Fixing syntax before the standard text is published. Refuted (not specific): Duplicate of standards-compat-2, which names a concrete alternative spelling the published edition could take.
- `replication-ondisk-2` Parse-time rewrite stores COALESCE in pg_rewrite permanently. Refuted (not specific): Duplicate of standards-compat-3. The stored COALESCE form gives the same results, so the added point about permanence describes no further failure.
- `maintenance-fit-2` Aggregate rewrites must each remember ON EMPTY, with no loud failure. Refuted (not specific): Duplicate of replication-ondisk-1, which names a concrete existing consumer of Aggref for the same class of silent drop.
- `maintenance-fit-4` Commits syntax before the standard edition is published. Refuted (not specific): Duplicate of standards-compat-2. The point about extending beyond SUM and PRODUCT is covered by standards-compat-1.
- `already-exists-1` ON EMPTY is a second spelling of COALESCE over the aggregate. Refuted (not specific): Duplicate of demand-1. Its migration point is covered by demand-2 and its view dump point by standards-compat-3.

### Review angles

- prior-attempts: 2 objection(s). Message #4 points to an earlier PRODUCT() thread where the standard's COALESCE definition was quoted. The material does not show how that thread ended.
- standards-compat: 3 objection(s). Three objections are raised, so no note is needed.
- replication-ondisk: 2 objection(s). The v1 to v3 catversion bump is routine, and pg_upgrade restores views from SQL, so old clusters are not at risk. The proposal has no WAL, standby, or wire protocol effect.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. ON EMPTY only substitutes a value for a NULL aggregate result, which any user can already do with COALESCE, and the material shows no new privileges, catalogs, maintenance-command execution paths, information channels or unbounded resource use.
- maintenance-fit: 4 objection(s). Objections found; committer Peter Eisentraut has not commented on them in the visible thread.
- already-exists: 1 objection(s). One objection from this angle. Existing coalesce() and case with count() already give every result the proposal offers.
- demand: 2 objection(s). Two demand objections are raised. The thread shows no field reports or independent user requests, so the fitting outcome is 'not now, show demand'.

### Precedents retrieved

- P1: [Partial aggregates push down](https://commitfest.postgresql.org/patch/3369/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P2: [onlyvalue aggregate](https://commitfest.postgresql.org/patch/407/) (cf, Returned with feedback, 2015-12-24) - judged not relevant; reason summarised from its thread
- P3: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4019/) (cf, Returned with feedback, 2024-01-27) - judged not relevant; reason summarised from its thread
- P4: [Add SQL syntax check](https://commitfest.postgresql.org/patch/4704/) (cf, Rejected, 2024-05-15) - judged not relevant; reason summarised from its thread
- P5: [empty](https://commitfest.postgresql.org/patch/5037/) (cf, Rejected, 2024-06-21) - judged not relevant
- P6: [Revert "Enable fast default for domains with non-volatile constraints"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0354e29c41a9fb7491b3c7c23f079b1923c045a) (revert, Reverted, 2026-06-08) - judged not relevant
- P7: [window function improvements WIP](https://commitfest.postgresql.org/patch/2419/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P8: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4857/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P9: [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (cf, Rejected, 2018-07-19) - used in the review; reason summarised from its thread
- P10: [Push down aggregates below joins.](https://commitfest.postgresql.org/patch/1685/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P11: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P12: [Flush disk write caches by default on macOS and Windows](https://commitfest.postgresql.org/patch/4453/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P13: [Revert "Optimize various aggregate deserialization functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f3b56eea23554e1756a26080db273156f23f4f2) (revert, Reverted, 2023-10-10) - judged not relevant
- P14: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P15: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P16: [skip empty transactions for logical replication](https://commitfest.postgresql.org/patch/2384/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P17: [Revert "Tweak "line" test to avoid negative zeros on some platforms"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f7a47912af2206698d14650f2149fa951b8ce07) (revert, Reverted, 2013-10-25) - judged not relevant
- P18: [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (cf, Returned with feedback, 2019-09-11) - judged not relevant
- P19: [Fix creation of empty .log files during log rotation](https://commitfest.postgresql.org/patch/5365/) (cf, Rejected, 2026-01-08) - judged not relevant
- P20: [Implement ASSERTIONs per the SQL standard.](https://commitfest.postgresql.org/patch/1604/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P21: [Boolean partition syntax](https://commitfest.postgresql.org/patch/1410/) (cf, Returned with feedback, 2018-03-21) - judged not relevant
- P22: [Implement <null treatment> for window functions](https://commitfest.postgresql.org/patch/2628/) (cf, Returned with feedback, 2021-03-25) - judged not relevant
- P23: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P24: [Check consistency of GUC defaults between .sample.conf and pg_settings.boot_val](https://commitfest.postgresql.org/patch/3736/) (cf, Returned with feedback, 2024-01-26) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 166 s; cost reported by the CLI: $2.43
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 122 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:10:18.465Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
