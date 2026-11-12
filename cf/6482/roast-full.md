# Design pre-review: Add table COMMENTS in CREATE TABLE LIKE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6482](https://commitfest.postgresql.org/patch/6482/) (Needs review); authors: Jim Jones (jimus)
- Thread: [COMMENTS are not being copied in CREATE TABLE LIKE](https://www.postgresql.org/message-id/flat/e08cb97f-0364-4002-9cda-3c16b42e4136%40uni-muenster.de), started 2026-02-12
- Messages read: 37 of 37
- Patch prose read: v5-0001-Add-table-comments-in-CREATE-TABLE-LIKE-INCLUDING.patch, v1-0001-Fix-missing-table-level-comment-copying-in-CREATE.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The decisive objection is demand-1: the thread shows no user or workflow that needs the table comment copied, and Tom Lane (message #34) and Tomas Vondra (message #33) asked for one. demand-2 adds that every `including all` user gets the new comment and has no exact spelling for the old result. Both are fixable and fit "not now, show demand". Design objections at level 2 were not evaluated.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the exact v5 behaviour for view, foreign table and composite type sources is unverified, as is how v5 counts LIKE clauses. The level 2 objections standards-compat-2 and standards-compat-3 survived but were not evaluated because level 1b fails first.

## The proposal, restated

**Problem.** Create table (like ... including comments), and so including all, copies comments on columns, constraints, indexes and extended statistics, but not the comment on the source table itself. The author found this while reviewing another patch and first called it a bug. Reviewers noted the documentation lists only those objects, so the change is a feature, and no user has asked for it.

**Proposed design.** v5 copies the source relation's comment to the new table under including comments when the command has exactly one like clause. With several like clauses, no table comment is copied, but comments on columns and other objects still are. v4 concatenated multiple table comments with newlines; v5 drops that.

**User-visible surface.**
- Changed behaviour of `including comments` in `create table ... (like ...)`: the source relation's own comment is copied.
- Changed behaviour of `including all`, which now also copies the table comment.
- Applies to `create foreign table ... (like ...)` too; v4 fixed an error in v3 for foreign table targets.
- A composite type used as a like source passes its type comment to the new table, as v4 regression tests show.

**What the author claims.**
- Copying makes including comments cover every comment on the objects that like copies.
- No single source comment describes a table built from several sources, so v5 skips that case.
- Users who do not want the table comment can write `including all excluding comments`.
- A reviewer reported tests passed and saw no performance change, including a case with hundreds of source tables.

**Thread state.** Tom Lane and Tomas Vondra question the lack of a real use case, and Tom would wait for field requests; the author then posted v5 without multi-table support and agreed the use case is the root issue. No one has replied to v5, and the patch has moved through four CommitFests.

## 2. Prior attempts

### create table like: ACCESS METHOD (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2865/](https://commitfest.postgresql.org/patch/2865/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20201209201328.GT24052@telsasoft.com](https://www.postgresql.org/message-id/flat/20201209201328.GT24052@telsasoft.com)
- **Date:** decided 2021-10-01; started 2020-12-09
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut, Michael Paquier, Vik Fearing (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** LIKE expands to column definitions, but the access method is a table property, so two LIKE clauses with different access methods can conflict. Its interaction with USING is also undefined. The patch overwrote any AM given in USING, and Michael Paquier asked for a clear definition of these corner cases. Peter Eisentraut argued that this syntax is the wrong approach and suggested something like USING (LIKE t1).
- **Reason (quote):** "Whereas the access method is a table property. So I don't think this syntax is the right approach for this feature." ([message](https://www.postgresql.org/message-id/8b530d40-1e85-e3b4-774f-b9cb4e2b7ca2%40enterprisedb.com))
- **What is different this time:** It copies the table comment and not the access method, so there is no clash with USING. v5 avoids the multi-LIKE conflict by copying no table comment when there are several LIKE clauses.

### Proposal to have INCLUDE/EXCLUDE options for altering option values (related)

- **Link:** [https://commitfest.postgresql.org/patch/4955/](https://commitfest.postgresql.org/patch/4955/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CACX+KaO1ZM4FA4k6jmpay4M+7qVvPG8VkbHGLQFfjNeF3sbv-w@mail.gmail.com](https://www.postgresql.org/message-id/flat/CACX+KaO1ZM4FA4k6jmpay4M+7qVvPG8VkbHGLQFfjNeF3sbv-w@mail.gmail.com)
- **Date:** decided 2024-10-08; started 2024-04-26
- **Outcome:** Withdrawn (cf)
- **Who objected:** Robert Haas, "Euler Taveira" (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Robert Haas and Euler Taveira doubted the use case was common enough to justify new syntax, since a user can read the option value, edit the list and SET it again. They also noted that most options are not lists and that list options need not be comma-separated. Euler Taveira added that the actions are not idempotent and that REMOVE is confusing next to DROP. The author agreed and withdrew the patch.
- **Reason (quote):** "Also, I'm not convinced that this problem would arise often enough in practice that it's worth adding a feature to address it." ([message](https://www.postgresql.org/message-id/CA%2BTgmob-MtTrr%3D9dYmXU-1tszc4wOUyj2zzd%2BM3AOV%3DkE6FhUA%40mail.gmail.com))
- **What is different this time:** It is a different feature that added new syntax for foreign server options. Only the doubt about real-world need is shared.


## 3. Objections a reviewer is likely to raise

### 1. No field request or workflow shows anyone needs the table comment copied

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer commits v5 to close a gap noticed during patch review. Later a real user turns up whose workflow needs different semantics, for example first-wins with several LIKE clauses or no copying of type comments. The project then has to change shipped behaviour of including comments a second time.

The patch started because the feature was missing (message #1), and no user report has appeared since. Tomas Vondra could not build a meaningful example (message #33), and Tom Lane would wait for field requests before choosing semantics (message #34). The author agreed that the missing use case is the root issue (messages #36, #37). Dropping multi-table support narrows the patch but does not add a user. The workaround, a comment on table statement after create table, costs one line, as in [P9].

**Precedents.**
- [Proposal to have INCLUDE/EXCLUDE options for altering option values](https://commitfest.postgresql.org/patch/4955/) (Withdrawn, 2024-10-08); thread: [link](https://www.postgresql.org/message-id/flat/CACX+KaO1ZM4FA4k6jmpay4M+7qVvPG8VkbHGLQFfjNeF3sbv-w@mail.gmail.com)

Thread: [#1 (Jim Jones, 2026-02-12)](https://www.postgresql.org/message-id/e08cb97f-0364-4002-9cda-3c16b42e4136%40uni-muenster.de), [#33 (Tomas Vondra, 2026-10-01)](https://www.postgresql.org/message-id/6cfde153-bb97-4fa5-a7e2-6b7c2056ae5d%40vondra.me), [#34 (Tom Lane, 2026-10-01)](https://www.postgresql.org/message-id/1113328.1790876738%40sss.pgh.pa.us), [#36 (Jim Jones, 2026-10-01)](https://www.postgresql.org/message-id/5885bc81-73d3-4c78-876f-5883659ce5f9%40uni-muenster.de), [#37 (Jim Jones, 2026-10-01)](https://www.postgresql.org/message-id/6f607039-c3c0-4b2a-96ff-4300eea85e39%40uni-muenster.de)

**Question:** Which users or tools today run create table like and then copy the table comment by hand, and what do they expect when several like clauses are used?

Counter-check: held. The thread shows no user request. Tom Lane (#34) and Tomas Vondra (#33) ask for one, and the author agrees in #36 and #37. [P9] concerned new syntax, so it is only weak support.

### 2. Changing including all affects existing scripts for an unproven gain

fixable · level 1b, rubric item 4 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-2`

A deployment script clones a template table commented 'template, do not use directly' with including all. After an upgrade, every derived table carries that comment. Users and catalog-browsing tools read it as a description of the new table. The script owner finds out only when someone asks why production tables are marked as templates.

Hüseyin Demir raised this in messages #11 and #13. The author's answer, including all excluding comments (message #14), also drops column, constraint and index comments, so the old result has no exact spelling. The script must instead add comment on table ... is null. Every including all user pays this cost, and the material names no user who gains.

Thread: [#11 (Hüseyin Demir, 2026-02-22)](https://www.postgresql.org/message-id/177174751973.626.7673368092428390674.pgcf%40coridan.postgresql.org), [#13 (Hüseyin Demir, 2026-02-26)](https://www.postgresql.org/message-id/CAB5wL7Yp3o4yYdxQ6mBVV6B-s80BmueizkKmZHEC632wYPp8Bw%40mail.gmail.com), [#14 (Jim Jones, 2026-02-26)](https://www.postgresql.org/message-id/3ecadefa-f63a-48a6-b732-b3e79814933b%40uni-muenster.de)

**Question:** What evidence shows that more including all users want the table comment copied than users who would now have to remove it?

Counter-check: held. Message #14 answers that any LIKE extension changes `including all`, but this patch also changes the existing `including comments` option, and `excluding comments` drops the column comments too. The cost is one extra statement, so fixable is the right severity.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch changes what `including comments` and `including all` mean in `create table ... (like ...)`. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #2 asked for a backpatch, but message #3 classes the change as an improvement, so it is for the next major version only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The single-LIKE rule and the set of source kinds that pass a comment become released behaviour to keep compatible. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A `comment on table` statement after `create table` already gives the same result in one line. |

## 5. Questions the author must answer before anyone reviews code

1. Which users or tools today run `create table ... (like ...)` and then copy the table comment by hand, and what do they expect with several LIKE clauses?
2. What evidence shows that more `including all` users want the table comment copied than users who would now have to remove it?
3. How does a user on the new release copy column, constraint and index comments but no table comment in one command?
4. Why should a second LIKE clause with `excluding all` stop the first clause from copying its table comment?
5. Which source kinds (table, view, foreign table, composite type) pass their comment to the new table in v5, and why is each one right?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane and Tomas Vondra took part (messages #33, #34), but both questioned the need and neither has replied to v5. The owner would be someone who knows the LIKE expansion code in parse_utilcmd.c.
- **Liability:** The owner signs for a permanent behaviour change of `including comments` and `including all`, for the single-LIKE rule, and for which source kinds pass their comment. The code itself is small.
- **Author follow-up:** The author posted five versions over about nine months, dropped multi-table support after feedback, and moved the entry between CommitFests.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A field report, a pgsql-general or pgsql-bugs thread, or a named tool that copies table comments after `create table ... (like ...)`, with the behaviour it expects for several LIKE clauses and for composite type sources. An argument that the change to `including all` hurts fewer users than it helps would also count.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** unclear
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 15 KiB in the parser, docs and regression tests, so it is cheap to review and carry. The thread names no user, and two committers asked to wait for a field request, so more review now does not move it forward.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Per-clause option whose effect depends on how many LIKE clauses exist. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-3` (fixable, level 2) Type and view comments become table comments without a decision. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` No field request shown, and v5 does not answer the request for one. Refuted (not specific): Duplicate of demand-1, which makes the same point from messages #33, #34 and #36 and also names the existing one-statement alternative.
- `prior-attempts-2` Table comment disappears when a second like clause is added. Refuted (not specific): Duplicate of standards-compat-2, which describes the same silent loss and adds that a clause with `excluding all` also triggers it.
- `standards-compat-1` No way to keep the old result except dropping all comments. Refuted (not specific): Duplicate of demand-2. Its claim that the old result cannot be expressed is too strong, because `comment on table ... is null` after the command restores it, as demand-2 says.
- `maintenance-fit-1` Single-LIKE rule makes the table comment depend on clause count. Refuted (not specific): Duplicate of standards-compat-2. Its extra point about later table-level INCLUDING options has no concrete failure scenario in the material.
- `maintenance-fit-2` Release fixes multi-LIKE behaviour before any use case exists. Refuted (already handled): With several LIKE clauses, v5 copies no table comment, which is what Postgres does today, so v5 fixes no new rule for that case. The remaining demand point is covered by demand-1.
- `already-exists-1` One comment on table statement already does what v5 automates. Refuted (not specific): Duplicate of demand-1, which already names the `comment on table` alternative. The cost to existing `including all` users is covered by demand-2.

### Review angles

- prior-attempts: 2 objection(s). Messages #10, #31 and #33 report that the 2009 thread adding including comments does not discuss table comments. Message #21 quotes a wish from that thread to clone all comments reasonably possible. [P3] has no recorded reason that applies.
- standards-compat: 3 objection(s). pg_dump round-trip is not affected: the dump records the resulting table and its comment, not the LIKE clause.
- replication-ondisk: 0 objection(s). The proposal adds no new persistent or cross-node state. The table comment is written to pg_description through the existing COMMENT path, with no catalog, WAL, on-disk, protocol or ABI change, so pg_upgrade, standbys and pg_dump see an ordinary comment.
- security: 0 objection(s). No security objection to v5. Comments in pg_description are readable by any role, so copying one into a new table owned by the creator exposes nothing new. The concatenation size concern from messages #31 and #35 no longer applies because v5 drops multi-source concatenation, and copying a single existing comment adds no unbounded allocation. Message #33 notes that setting a comment already requires ownership, so the patch opens no new path for an untrusted role to change what the copy produces.
- maintenance-fit: 2 objection(s). The patch touches only the parser and docs, has no hot-path cost, and adds no external dependency. Maintenance cost is small apart from the design points above.
- already-exists: 1 objection(s). One objection from this angle. No contrib module or extension covers this, so the existing way is plain SQL plus psql \gexec or a do block.
- demand: 2 objection(s). Two demand objections. The material shows no user request, and the change of including all falls on users who did not ask for it.

### Precedents retrieved

- P1: [create table like: ACCESS METHOD](https://commitfest.postgresql.org/patch/2865/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P2: [Revert "Distinguish printf-like functions that support %m from those that don't."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=46b5e7c4b5befbf6ac86d827a3a58f1f02c7338e) (revert, Reverted, 2018-08-12) - judged not relevant
- P3: [Revert: Honor OID status of CREATE LIKE'd tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=764ce22af30df3d62524d41815c41b4d696f3c10) (revert, Reverted, 2015-04-26) - used in the review
- P4: [Revert support for ALTER TABLE ... MERGE/SPLIT PARTITION(S) commands](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3890d90c1508125729ed20038d90513694fc3a7b) (revert, Reverted, 2024-08-24) - judged not relevant
- P5: [Create shorthand for including all extra tests](https://commitfest.postgresql.org/patch/4547/) (cf, Rejected, 2024-01-20) - judged not relevant; reason summarised from its thread
- P6: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P7: [Remove unused #include's](https://commitfest.postgresql.org/patch/6112/) (cf, Withdrawn, 2025-10-15) - judged not relevant; reason summarised from its thread
- P8: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P9: [Proposal to have INCLUDE/EXCLUDE options for altering option values](https://commitfest.postgresql.org/patch/4955/) (cf, Withdrawn, 2024-10-08) - used in the review; reason summarised from its thread
- P10: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - judged not relevant
- P11: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P12: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant; reason summarised from its thread
- P13: [Revert "Propagate CTE property flags when copying a CTE list into a rule."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d2979852538d7021cc809a40ef127d59747697) (revert, Reverted, 2021-02-07) - judged not relevant
- P14: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant; reason summarised from its thread
- P15: [A separate table level option to control compression](https://commitfest.postgresql.org/patch/2001/) (cf, Returned with feedback, 2019-04-08) - judged not relevant
- P16: [Reduce lock level for ALTER TABLE ... ADD CHECK](https://commitfest.postgresql.org/patch/3088/) (cf, Withdrawn, 2021-11-21) - judged not relevant
- P17: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant
- P18: [Revert "Remove reset of testtablespace from pg_regress on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61be85afabba0cd85bb1bcaacbea8efa8641f564) (revert, Reverted, 2020-07-10) - judged not relevant
- P19: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P20: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P21: [Revert to the pre-8.2 method of probing for libm, that is, always include it if it links properly.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=02f37bd8c4bfdd35869007d142267cd9a1f671b9) (revert, Reverted, 2006-11-06) - judged not relevant
- P22: [Reduce lock levels for ALTER TABLE reloptions](https://commitfest.postgresql.org/patch/783/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P23: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P24: [Parallel Inserts in CREATE TABLE AS](https://commitfest.postgresql.org/patch/2841/) (cf, Withdrawn, 2021-03-20) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 124 s; cost reported by the CLI: $2.98
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 102 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:53:35.506Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
