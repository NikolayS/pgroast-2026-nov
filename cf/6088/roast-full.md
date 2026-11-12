# Design pre-review: create table like including storage parameter

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6088](https://commitfest.postgresql.org/patch/6088/) (Needs review); authors: Jian He (stayreal)
- Thread: [create table like including storage parameter](https://www.postgresql.org/message-id/flat/CACJufxHr%3DnKEsS66M7rTHgB%2BmXdQ948%3D_eJ1jztAc7PT-eJefA%40mail.gmail.com), started 2025-09-29
- Messages read: 17 of 17
- Patch prose read: v10-0001-Add-INCLUDING-PARAMETERS-to-CREATE-TABLE-LIKE.patch, v1-0001-create-table-like-incluing-storage-parameter.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is reasonable, and the open point is how copied reloptions combine with other sources. Per standards-compat-1, folding PARAMETERS into including all makes commands that work today fail with a duplicate parameter error. Per standards-compat-2, the proposal gives no rule for several like sources plus a with clause, the same gap that stalled [P1]. The demand objection (demand-1) is real but would not stop a committer by itself.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the reloptions.h change, the handling of toast.* options, and the exact behaviour for equal values from two sources cannot be checked. Only 3 of 17 messages were read directly.

## The proposal, restated

**Problem.** create table ... like copies columns, defaults, constraints, indexes and more, but not the source table's storage parameters (pg_class.reloptions). Users who want a similar table must repeat every storage parameter by hand.

**Proposed design.** A new like_option PARAMETERS copies the source relation's reloptions, including toast.* options, to the new table, and including all now covers it. Parameters that are also given in the with clause raise a duplicate error. The option is ignored when the source is a view, for create foreign table ... like, and for a partitioned target table.

**User-visible surface.**
- New like_option in create table: { including | excluding } parameters.
- including all now also copies table storage parameters.
- New keyword PARAMETERS added to the grammar keyword list.
- Changed C API in reloptions.h, per the touched-file list; the material does not say what changed.

**What the author claims.**
- The option saves users from listing every storage parameter by hand.
- It makes a table made with like closer to the source table.
- Earlier version notes say it applies to matviews as well as tables.
- A failure from conflicting parameters across two like clauses is no different from other existing like failures.

**Thread state.** David Johnston and Nathan Bossart chose the PARAMETERS name, Euler Taveira reviewed the code, and lakshmi tested v9. Fujii Masao's backward compatibility concern about including all with two sources is open after the author's reply.

## 2. Prior attempts

### create table like: ACCESS METHOD (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2865/](https://commitfest.postgresql.org/patch/2865/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20201209201328.GT24052@telsasoft.com](https://www.postgresql.org/message-id/flat/20201209201328.GT24052@telsasoft.com)
- **Date:** decided 2021-10-01; started 2020-12-09
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut, Michael Paquier, Vik Fearing (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** LIKE expands to column definitions, but the access method is a table property, so two LIKE clauses with different access methods can conflict. Its interaction with USING is also undefined. The patch overwrote any AM given in USING, and Michael Paquier asked for a clear definition of these corner cases. Peter Eisentraut argued that this syntax is the wrong approach and suggested something like USING (LIKE t1).
- **Reason (quote):** "Whereas the access method is a table property. So I don't think this syntax is the right approach for this feature." ([message](https://www.postgresql.org/message-id/8b530d40-1e85-e3b4-774f-b9cb4e2b7ca2%40enterprisedb.com))
- **What is different this time:** P1 copied the access method, and this proposal copies reloptions. This one raises an error on a conflict with the with clause, where P1 overwrote USING. It still gives no rule for two like sources.


## 3. Objections a reviewer is likely to raise

### 1. Including all now fails on commands that succeed today

fixable · level 2, rubric item 6 · SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-1`

A deployment script runs create table x (like t1 including all, like t2 including all), and both sources set fillfactor. Another runs create table y (like t including all) with (fillfactor = 90) on a source that already sets fillfactor. Both work on 19. After upgrade both fail with a duplicate parameter error, and the user finds out only when the migration breaks.

Message #16 raises the two-source case, and the brief notes it is still open. Message #17 answers that like can already fail, for example on duplicate columns, but that failure exists today, while this one is new for unchanged commands. Message #1 and the commit message make a parameter given in both with and like an error. The proposal does not consider letting with override copied values, or merging equal values, which would keep existing scripts working.

Thread: [#1 (jian he, 2025-09-29)](https://www.postgresql.org/message-id/CACJufxHr%3DnKEsS66M7rTHgB%2BmXdQ948%3D_eJ1jztAc7PT-eJefA%40mail.gmail.com), [#16 (Fujii Masao, 2026-10-06)](https://www.postgresql.org/message-id/CAHGQGwEBB%3Dc-7psc6wB2bAvzJPQ3-Nrz1WR-LV4V0kYRdScr4A%40mail.gmail.com), [#17 (jian he, 2026-10-12)](https://www.postgresql.org/message-id/CACJufxF5ivkQnGtTruK6GfpvLMHy%3DNH-YMATPqEhdVtu7k6qkQ%40mail.gmail.com)

**Question:** Why should copied reloptions raise an error instead of yielding to an explicit with clause or to an equal value from another source?

Counter-check: held. Message #1 confirms the duplicate error, and message #17 cites only a failure that exists today. Earlier like options also joined including all, but none made a working command fail.

### 2. A table-level property copied through a per-source like clause

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax · `standards-compat-2`

A user writes create table x (like hot_table including parameters, like cold_table including parameters), where the two sources set different fillfactor and autovacuum settings. The command fails. The user can only fix it by switching to excluding parameters on one source, and nothing in the docs says which source should win.

Like expands into column-level elements, but reloptions belong to the whole table. [P1] stalled for the same reason with access methods, because two like clauses could conflict and the interaction with USING was not defined. Here the with clause has the same role as USING there. The v10 documentation hunk does not describe conflicts between sources, between a source and with, or the cases where the option is ignored.

**Precedents.**
- [create table like: ACCESS METHOD](https://commitfest.postgresql.org/patch/2865/) (Returned with feedback, 2021-10-01); thread: [link](https://www.postgresql.org/message-id/flat/20201209201328.GT24052@telsasoft.com)

Thread: [#15 (jian he, 2026-09-29)](https://www.postgresql.org/message-id/CACJufxG4w4hdM6Bu3pZNV2nOGbjMYiCmAN%3Dzfh0BeGK-Dg7hvA%40mail.gmail.com), [#16 (Fujii Masao, 2026-10-06)](https://www.postgresql.org/message-id/CAHGQGwEBB%3Dc-7psc6wB2bAvzJPQ3-Nrz1WR-LV4V0kYRdScr4A%40mail.gmail.com)

**Question:** What rule decides the outcome when several like sources and the with clause set the same parameter, and where is it documented?

Counter-check: held. [P1] fits: reloptions, like an access method, are single-valued table properties that can conflict across like clauses and with an explicit clause. The thread does not say why the like syntax suits such a property, or what happens with equal values.

### 3. No user demand shown beyond convenience for one workflow

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), SQL-standard conflict or non-standard syntax · `demand-1`

The project adds a new keyword and like_option to support it for years. Users who clone tables with custom reloptions already copy pg_class.reloptions into a with clause, or run alter table ... set afterwards. Few people notice the new option, and its cost stays.

No message cites a field report, a -general or -bugs request, or another system that offers this. The case rests on the commit message's claim of convenience, and messages #2 and #5 discuss naming, not need. Reloptions are few and easy to read from pg_class, so a short script covers the gap. A committer would likely ask who has been blocked by this.

Thread: [#1 (jian he, 2025-09-29)](https://www.postgresql.org/message-id/CACJufxHr%3DnKEsS66M7rTHgB%2BmXdQ948%3D_eJ1jztAc7PT-eJefA%40mail.gmail.com), [#2 ("David G(dot) Johnston", 2025-09-29)](https://www.postgresql.org/message-id/CAKFQuwbFTgLv4t7f63Xsco2r_8qjW9bdpi5vg1nq0D-o%3DX_6GQ%40mail.gmail.com), [#5 (Nathan Bossart, 2025-10-24)](https://www.postgresql.org/message-id/aPu77xiPRJaIlc49%40nathan), [#9 (jian he, 2025-12-16)](https://www.postgresql.org/message-id/CACJufxG3N8mNXjtP_VDR1V54kfinU3jrEvCjpUO-n056hnOqVA%40mail.gmail.com)

**Question:** Which users or tools clone tables with like today and hit missing reloptions, and why is reading pg_class.reloptions not enough for them?

Counter-check: held. The thread shows no user report or request, and a with clause or alter table ... set does the job today. Two committers engaged without questioning the need, which lowers the weight but does not show demand.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes src/include/access/reloptions.h, and the material does not say what extensions will see. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds a non-standard like_option and a new keyword PARAMETERS to the grammar. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Including all changes meaning for unchanged commands, so some scripts that work on the old release fail after upgrade. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new keyword, a conflict rule, and skip rules for views, foreign tables and partitioned targets must be kept in step with reloptions. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Users can already copy pg_class.reloptions into a with clause or run alter table ... set afterwards. |

## 5. Questions the author must answer before anyone reviews code

1. What is the result when two like sources set the same parameter, with equal values and with different values?
2. Why does a parameter given in the with clause raise an error instead of overriding the value copied by like?
3. Why must including all cover PARAMETERS, when users who want reloptions copied can write including parameters?
4. Which users or tools clone tables with like today and are blocked by missing reloptions?
5. What changes in reloptions.h, and does it affect extensions that call the existing reloptions functions?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Someone who knows the create table like parse code and reloptions. Fujii Masao raised the open compatibility concern in message #16, and Nathan Bossart removed himself as reviewer in the CommitFest history.
- **Liability:** The owner signs for the new keyword, the conflict rule, and the changed meaning of including all, with no end date because scripts depend on it.
- **Author follow-up:** The author posted ten versions over about a year, kept the entry rebased across six CommitFests, and replied to the concern in message #17.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 19 KiB in the parser and reloptions code and adds one keyword, but the thread shows no user request. The conflict rule and the including all behaviour need agreement before more code review helps.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Including all plus a with clause now fails for existing commands. Refuted (not specific): Duplicate of standards-compat-1, which covers the with clause case and the two-source case without relying on [P1].
- `prior-attempts-2` Two like sources with the same parameter break a working command. Refuted (not specific): Duplicate of standards-compat-1, which states the same two-source regression from message #16 and the same gap in the reply in message #17.
- `standards-compat-3` Generic name PARAMETERS invites later changes to including all. Refuted (no failure scenario): The scenario needs a later patch that copies column options, and message #11 already argued against that. This patch copies only pg_class.reloptions, and any wider scope would get its own review.
- `maintenance-fit-1` Including all starts copying reloptions, so existing commands can fail. Refuted (not specific): Duplicate of standards-compat-1. Its extra question about leaving parameters out of including all belongs with that objection.
- `maintenance-fit-2` Per-relkind silent ignore rules will drift as reloptions change. Refuted (no failure scenario): The scenario depends on a future change, such as reloptions for partitioned tables or a new relkind, that the proposal does not include. The missing documentation of the ignore cases is a docs matter, outside levels 1-2.
- `demand-2` Including all users pay for a gain only some of them want. Refuted (not specific): Duplicate of standards-compat-1: same two failing commands, same reading of messages #16 and #17. The who-pays framing adds no new scenario.

### Review angles

- prior-attempts: 2 objection(s). Both objections come from P1's recorded reason. P2 is a revert whose recorded reason gives no detail, so it is not used.
- standards-compat: 3 objection(s). pg_dump does not emit like clauses, so the dump round-trip is not affected. Silently ignoring the option for views and foreign tables follows how other like options already behave.
- replication-ondisk: 0 objection(s). The proposal only copies existing reloptions into the new table's pg_class.reloptions at create time, so it adds no on-disk format, WAL record, catalog column or wire protocol change, and pg_dump does not emit LIKE. The touched-file list shows a change to reloptions.h, but the material does not say what changed, so whether extensions that call untransformRelOptions() are affected cannot be checked without the code.
- security: 0 objection(s). No security objection. The source table's reloptions are already readable by any user through pg_class, and like already requires access to the source table. Every copied storage parameter is one the new table's owner could set in a with clause, so the option grants no new privilege, exposes no new data and opens no new denial-of-service path.
- maintenance-fit: 2 objection(s). Two objections. Demand and fit look acceptable, and several reviewers engaged.
- already-exists: 0 objection(s). Today a user can read pg_class.reloptions for the source table and its TOAST table, then run alter table ... set (...) or build the with clause in a DO block. That is less convenient but works, and the other like_options (COMMENTS, STORAGE, COMPRESSION, STATISTICS) also copy properties that users could copy by hand. No contrib module or extension does this, and no hook would be cheaper than the like_option itself. The conflict between two like sources and the with clause, and the table-property concern in [P1], belong to the design angle.
- demand: 2 objection(s). Two demand-side objections. Neither is fatal: both ask for evidence or a narrower default.

### Precedents retrieved

- P1: [create table like: ACCESS METHOD](https://commitfest.postgresql.org/patch/2865/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P2: [Revert: Honor OID status of CREATE LIKE'd tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=764ce22af30df3d62524d41815c41b4d696f3c10) (revert, Reverted, 2015-04-26) - used in the review
- P3: [Create shorthand for including all extra tests](https://commitfest.postgresql.org/patch/4547/) (cf, Rejected, 2024-01-20) - judged not relevant; reason summarised from its thread
- P4: [Revert CREATE INDEX ... INCLUDING ...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b99edefcab1e82c43139a2c7dc06d31fb27b3e4) (revert, Reverted, 2016-04-08) - judged not relevant
- P5: [Reloptions for table access methods](https://commitfest.postgresql.org/patch/2717/) (cf, Withdrawn, 2021-04-06) - judged not relevant; reason summarised from its thread
- P6: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P7: [Revert "Custom reloptions for table AM"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=867cc7b6ddb9d998103688a56048fe9a1ddd972a) (revert, Reverted, 2024-04-02) - judged not relevant
- P8: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - judged not relevant
- P9: [Revert: Custom reloptions for table AM](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bc1e2092ebb857802a9713d0d3588079e2f0216a) (revert, Reverted, 2024-04-11) - judged not relevant
- P10: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P11: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant; reason summarised from its thread
- P12: [Parallel Inserts in CREATE TABLE AS](https://commitfest.postgresql.org/patch/2841/) (cf, Withdrawn, 2021-03-20) - judged not relevant; reason summarised from its thread
- P13: [Reduce lock levels for ALTER TABLE reloptions](https://commitfest.postgresql.org/patch/783/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P14: [remove C++ keywords](https://commitfest.postgresql.org/patch/1075/) (cf, Returned with feedback, 2017-04-06) - judged not relevant; reason summarised from its thread
- P15: [Custom storage managers (SMGR), redux](https://commitfest.postgresql.org/patch/4428/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P16: [pgbench: faster version of tpcb-like transaction](https://commitfest.postgresql.org/patch/1257/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P17: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant
- P18: [\[doc\] vacuum full, fillfactor, and "extra space"](https://commitfest.postgresql.org/patch/2412/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P19: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - judged not relevant
- P20: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P21: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P22: [Columnar storage](https://commitfest.postgresql.org/patch/354/) (cf, Returned with feedback, 2016-01-31) - judged not relevant
- P23: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant
- P24: [Fillfactor for GIN indexes](https://commitfest.postgresql.org/patch/13/) (cf, Rejected, 2015-01-16) - judged not relevant

### Warnings

- review angle "standards-compat" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 155 s; cost reported by the CLI: $1.95
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 93 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:41:30.525Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
