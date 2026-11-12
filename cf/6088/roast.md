# Design pre-review: create table like including storage parameter

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6088](https://commitfest.postgresql.org/patch/6088/); [thread](https://www.postgresql.org/message-id/flat/CACJufxHr%3DnKEsS66M7rTHgB%2BmXdQ948%3D_eJ1jztAc7PT-eJefA%40mail.gmail.com), 17 of 17 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is reasonable, and the open point is how copied reloptions combine with other sources. Per standards-compat-1, folding PARAMETERS into including all makes commands that work today fail with a duplicate parameter error. Per standards-compat-2, the proposal gives no rule for several like sources plus a with clause, the same gap that stalled [P1]. The demand objection (demand-1) is real but would not stop a committer by itself.

Not checked: Without the code, the reloptions.h change, the handling of toast.* options, and the exact behaviour for equal values from two sources cannot be checked. Only 3 of 17 messages were read directly.

## Prior attempts

- [create table like: ACCESS METHOD](https://www.postgresql.org/message-id/flat/20201209201328.GT24052@telsasoft.com) (similar idea; Returned with feedback, 2021-10-01). "Whereas the access method is a table property. So I don't think this syntax is the right approach for this feature." ([message](https://www.postgresql.org/message-id/8b530d40-1e85-e3b4-774f-b9cb4e2b7ca2%40enterprisedb.com)) Different now: P1 copied the access method, and this proposal copies reloptions. This one raises an error on a conflict with the with clause, where P1 overwrote USING. It still gives no rule for two like sources.

## Objections a reviewer is likely to raise

### 1. Including all now fails on commands that succeed today

A deployment script runs create table x (like t1 including all, like t2 including all), and both sources set fillfactor. Another runs create table y (like t including all) with (fillfactor = 90) on a source that already sets fillfactor. Both work on 19. After upgrade both fail with a duplicate parameter error, and the user finds out only when the migration breaks.

Message #16 raises the two-source case, and the brief notes it is still open. Message #17 answers that like can already fail, for example on duplicate columns, but that failure exists today, while this one is new for unchanged commands. Message #1 and the commit message make a parameter given in both with and like an error. The proposal does not consider letting with override copied values, or merging equal values, which would keep existing scripts working.

**Question:** Why should copied reloptions raise an error instead of yielding to an explicit with clause or to an equal value from another source?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/CACJufxHr%3DnKEsS66M7rTHgB%2BmXdQ948%3D_eJ1jztAc7PT-eJefA%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/CAHGQGwEBB%3Dc-7psc6wB2bAvzJPQ3-Nrz1WR-LV4V0kYRdScr4A%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CACJufxF5ivkQnGtTruK6GfpvLMHy%3DNH-YMATPqEhdVtu7k6qkQ%40mail.gmail.com); `standards-compat-1`</sub>

### 2. A table-level property copied through a per-source like clause

A user writes create table x (like hot_table including parameters, like cold_table including parameters), where the two sources set different fillfactor and autovacuum settings. The command fails. The user can only fix it by switching to excluding parameters on one source, and nothing in the docs says which source should win.

Like expands into column-level elements, but reloptions belong to the whole table. [P1] stalled for the same reason with access methods, because two like clauses could conflict and the interaction with USING was not defined. Here the with clause has the same role as USING there. The v10 documentation hunk does not describe conflicts between sources, between a source and with, or the cases where the option is ignored.

Precedent: [create table like: ACCESS METHOD](https://www.postgresql.org/message-id/flat/20201209201328.GT24052@telsasoft.com) (Returned with feedback, 2021-10-01)

**Question:** What rule decides the outcome when several like sources and the with clause set the same parameter, and where is it documented?

<sub>fixable, level 2, design; messages [#15](https://www.postgresql.org/message-id/CACJufxG4w4hdM6Bu3pZNV2nOGbjMYiCmAN%3Dzfh0BeGK-Dg7hvA%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/CAHGQGwEBB%3Dc-7psc6wB2bAvzJPQ3-Nrz1WR-LV4V0kYRdScr4A%40mail.gmail.com); `standards-compat-2`</sub>

### 3. No user demand shown beyond convenience for one workflow

The project adds a new keyword and like_option to support it for years. Users who clone tables with custom reloptions already copy pg_class.reloptions into a with clause, or run alter table ... set afterwards. Few people notice the new option, and its cost stays.

No message cites a field report, a -general or -bugs request, or another system that offers this. The case rests on the commit message's claim of convenience, and messages #2 and #5 discuss naming, not need. Reloptions are few and easy to read from pg_class, so a short script covers the gap. A committer would likely ask who has been blocked by this.

**Question:** Which users or tools clone tables with like today and hit missing reloptions, and why is reading pg_class.reloptions not enough for them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CACJufxHr%3DnKEsS66M7rTHgB%2BmXdQ948%3D_eJ1jztAc7PT-eJefA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAKFQuwbFTgLv4t7f63Xsco2r_8qjW9bdpi5vg1nq0D-o%3DX_6GQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/aPu77xiPRJaIlc49%40nathan), [#9](https://www.postgresql.org/message-id/CACJufxG3N8mNXjtP_VDR1V54kfinU3jrEvCjpUO-n056hnOqVA%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. What is the result when two like sources set the same parameter, with equal values and with different values?
2. Why does a parameter given in the with clause raise an error instead of overriding the value copied by like?
3. Why must including all cover PARAMETERS, when users who want reloptions copied can write including parameters?
4. Which users or tools clone tables with like today and are blocked by missing reloptions?
5. What changes in reloptions.h, and does it affect extensions that call the existing reloptions functions?

## Touch points

- **ABI / extension API:** The patch changes src/include/access/reloptions.h, and the material does not say what extensions will see.
- **SQL-standard conflict or non-standard syntax:** The patch adds a non-standard like_option and a new keyword PARAMETERS to the grammar.
- **backpatchability:** Including all changes meaning for unchanged commands, so some scripts that work on the old release fail after upgrade.
- **maintenance burden ("who maintains this in 10 years"):** A new keyword, a conflict rule, and skip rules for views, foreign tables and partitioned targets must be kept in step with reloptions.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Users can already copy pg_class.reloptions into a with clause or run alter table ... set afterwards.

## Who would commit it, and when

- Owner: qualified committer engaged. Someone who knows the create table like parse code and reloptions. Fujii Masao raised the open compatibility concern in message #16, and Nathan Bossart removed himself as reviewer in the CommitFest history.
- Priority: fix design first; audience niche; review cost low. The patch is about 19 KiB in the parser and reloptions code and adds one keyword, but the thread shows no user request. The conflict rule and the including all behaviour need agreement before more code review helps.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
