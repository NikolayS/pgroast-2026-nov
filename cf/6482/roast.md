# Design pre-review: Add table COMMENTS in CREATE TABLE LIKE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6482](https://commitfest.postgresql.org/patch/6482/); [thread](https://www.postgresql.org/message-id/flat/e08cb97f-0364-4002-9cda-3c16b42e4136%40uni-muenster.de), 37 of 37 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The decisive objection is demand-1: the thread shows no user or workflow that needs the table comment copied, and Tom Lane (message #34) and Tomas Vondra (message #33) asked for one. demand-2 adds that every `including all` user gets the new comment and has no exact spelling for the old result. Both are fixable and fit "not now, show demand". Design objections at level 2 were not evaluated.

Not checked: The code was not read, so the exact v5 behaviour for view, foreign table and composite type sources is unverified, as is how v5 counts LIKE clauses. The level 2 objections standards-compat-2 and standards-compat-3 survived but were not evaluated because level 1b fails first.

## Prior attempts

- [create table like: ACCESS METHOD](https://www.postgresql.org/message-id/flat/20201209201328.GT24052@telsasoft.com) (similar idea; Returned with feedback, 2021-10-01). "Whereas the access method is a table property. So I don't think this syntax is the right approach for this feature." ([message](https://www.postgresql.org/message-id/8b530d40-1e85-e3b4-774f-b9cb4e2b7ca2%40enterprisedb.com)) Different now: It copies the table comment and not the access method, so there is no clash with USING. v5 avoids the multi-LIKE conflict by copying no table comment when there are several LIKE clauses.
- [Proposal to have INCLUDE/EXCLUDE options for altering option values](https://www.postgresql.org/message-id/flat/CACX+KaO1ZM4FA4k6jmpay4M+7qVvPG8VkbHGLQFfjNeF3sbv-w@mail.gmail.com) (related; Withdrawn, 2024-10-08). "Also, I'm not convinced that this problem would arise often enough in practice that it's worth adding a feature to address it." ([message](https://www.postgresql.org/message-id/CA%2BTgmob-MtTrr%3D9dYmXU-1tszc4wOUyj2zzd%2BM3AOV%3DkE6FhUA%40mail.gmail.com)) Different now: It is a different feature that added new syntax for foreign server options. Only the doubt about real-world need is shared.

## Objections a reviewer is likely to raise

### 1. No field request or workflow shows anyone needs the table comment copied

A committer commits v5 to close a gap noticed during patch review. Later a real user turns up whose workflow needs different semantics, for example first-wins with several LIKE clauses or no copying of type comments. The project then has to change shipped behaviour of including comments a second time.

The patch started because the feature was missing (message #1), and no user report has appeared since. Tomas Vondra could not build a meaningful example (message #33), and Tom Lane would wait for field requests before choosing semantics (message #34). The author agreed that the missing use case is the root issue (messages #36, #37). Dropping multi-table support narrows the patch but does not add a user. The workaround, a comment on table statement after create table, costs one line, as in [P9].

Precedent: [Proposal to have INCLUDE/EXCLUDE options for altering option values](https://www.postgresql.org/message-id/flat/CACX+KaO1ZM4FA4k6jmpay4M+7qVvPG8VkbHGLQFfjNeF3sbv-w@mail.gmail.com) (Withdrawn, 2024-10-08)

**Question:** Which users or tools today run create table like and then copy the table comment by hand, and what do they expect when several like clauses are used?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/e08cb97f-0364-4002-9cda-3c16b42e4136%40uni-muenster.de), [#33](https://www.postgresql.org/message-id/6cfde153-bb97-4fa5-a7e2-6b7c2056ae5d%40vondra.me), [#34](https://www.postgresql.org/message-id/1113328.1790876738%40sss.pgh.pa.us), [#36](https://www.postgresql.org/message-id/5885bc81-73d3-4c78-876f-5883659ce5f9%40uni-muenster.de), [#37](https://www.postgresql.org/message-id/6f607039-c3c0-4b2a-96ff-4300eea85e39%40uni-muenster.de); `demand-1`</sub>

### 2. Changing including all affects existing scripts for an unproven gain

A deployment script clones a template table commented 'template, do not use directly' with including all. After an upgrade, every derived table carries that comment. Users and catalog-browsing tools read it as a description of the new table. The script owner finds out only when someone asks why production tables are marked as templates.

Hüseyin Demir raised this in messages #11 and #13. The author's answer, including all excluding comments (message #14), also drops column, constraint and index comments, so the old result has no exact spelling. The script must instead add comment on table ... is null. Every including all user pays this cost, and the material names no user who gains.

**Question:** What evidence shows that more including all users want the table comment copied than users who would now have to remove it?

<sub>fixable, level 1b, desirability; messages [#11](https://www.postgresql.org/message-id/177174751973.626.7673368092428390674.pgcf%40coridan.postgresql.org), [#13](https://www.postgresql.org/message-id/CAB5wL7Yp3o4yYdxQ6mBVV6B-s80BmueizkKmZHEC632wYPp8Bw%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/3ecadefa-f63a-48a6-b732-b3e79814933b%40uni-muenster.de); `demand-2`</sub>

## Questions for the author

1. Which users or tools today run `create table ... (like ...)` and then copy the table comment by hand, and what do they expect with several LIKE clauses?
2. What evidence shows that more `including all` users want the table comment copied than users who would now have to remove it?
3. How does a user on the new release copy column, constraint and index comments but no table comment in one command?
4. Why should a second LIKE clause with `excluding all` stop the first clause from copying its table comment?
5. Which source kinds (table, view, foreign table, composite type) pass their comment to the new table in v5, and why is each one right?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The patch changes what `including comments` and `including all` mean in `create table ... (like ...)`.
- **backpatchability:** Message #2 asked for a backpatch, but message #3 classes the change as an improvement, so it is for the next major version only.
- **maintenance burden ("who maintains this in 10 years"):** The single-LIKE rule and the set of source kinds that pass a comment become released behaviour to keep compatible.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A `comment on table` statement after `create table` already gives the same result in one line.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane and Tomas Vondra took part (messages #33, #34), but both questioned the need and neither has replied to v5. The owner would be someone who knows the LIKE expansion code in parse_utilcmd.c.
- Not now, show demand first: A field report, a pgsql-general or pgsql-bugs thread, or a named tool that copies table comments after `create table ... (like ...)`, with the behaviour it expects for several LIKE clauses and for composite type sources. An argument that the change to `including all` hurts fewer users than it helps would also count.
- Priority: park; audience unclear; review cost low. The patch is about 15 KiB in the parser, docs and regression tests, so it is cheap to review and carry. The thread names no user, and two committers asked to wait for a field request, so more review now does not move it forward.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
