# Design pre-review: gist_trgm_ops '=' operator: planner picks it over btree, ~300x slower

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7401](https://commitfest.postgresql.org/patch/7401/); [thread](https://www.postgresql.org/message-id/flat/CA%2BvB%3DAHon6EB%3D90pbPDik_u8%2B%2B%2BTy8TA6Z4-RXo_X5QCr71kcQ%40mail.gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is real and reproducible, and the patch costs non-users nothing. The thread has not settled whether a second opclass is the right fix. maintenance-fit-1 shows that Korotkov's point about the generic GiST estimator (message #4) is unanswered. standards-compat-1 shows that the opclass cannot be removed once indexes and dumps name it. Confidence is medium because only 3 of 10 messages were visible.

Not checked: Without the code I cannot confirm that gist_trgm_ops_noeq has its own opfamily, which keeping = away from the planner requires. Messages #4 to #10 were not visible, so the claims about them rest on the objections and the brief.

## Prior attempts

None found before 2026-09-02.

## Objections a reviewer is likely to raise

### 1. Opclass twin works around a generic GiST costing gap

A user with btree_gist, or another opclass whose = or other operators are costed like btree, hits the same plan flip. Once gist_trgm_ops_noeq exists, the next fix proposed for that opclass is another "_noeq" twin. Each one adds names that users must know about, while gistcostestimate stays unchanged.

In message #4 Korotkov says GiST cost for = should already be above btree under the generic estimator, because GiST calls the consistent function for every item on a page. That points to a gistcostestimate change that would help every opclass. The thread never answers this point, and message #6 compares patch 3 only with the ANALYZE hook. A committer would likely ask whether ten such per-opclass twins per release would be acceptable.

**Question:** Why does a per-item consistent-call charge in gistcostestimate not fix this case, and which other opclasses would then not need a similar twin?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/CAPpHfdtGzomdU%2B0wiwrZS52NRQYwMrzDraE0eCfwAM2RvQAcag%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CA%2BvB%3DAG1P9FMm30ov_w4%2B%2ByNZ9Cdy40hYdC0tLxxPT2GEQaQxQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Workaround opclass name becomes a permanent part of every dump

A user builds indexes with gist_trgm_ops_noeq in Postgres 20. A later release fixes GiST cost estimation for =, and pg_trgm wants to drop the extra opclass. Every dump and pg_upgrade from older clusters names it in CREATE INDEX, so removing it breaks restores, and the opclass stays for good.

Message #3 calls an ANALYZE-time hook the real fix, and message #4 argues that even the generic GiST estimator should charge more for =. Either fix would make this opclass redundant. The opclass name encodes a planner workaround in user DDL, and pg_dump writes it out for every index that uses it. The proposal gives no deprecation or retirement plan and does not say what happens to it once the cost model is fixed.

**Question:** If GiST cost estimation for = is fixed later, how would gist_trgm_ops_noeq be retired without breaking dumps and pg_upgrade of existing indexes?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#3](https://www.postgresql.org/message-id/69c88bb3-d9a8-4054-9024-1f324d31e5aa%40dunslane.net), [#4](https://www.postgresql.org/message-id/CAPpHfdtGzomdU%2B0wiwrZS52NRQYwMrzDraE0eCfwAM2RvQAcag%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CA%2BvB%3DAG1P9FMm30ov_w4%2B%2ByNZ9Cdy40hYdC0tLxxPT2GEQaQxQ%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Users can already define this opclass themselves with plain SQL

A user hit by the slow = plan can run `create operator class` today, reuse pg_trgm's support functions, leave out operator 11, and reindex. If pg_trgm ships gist_trgm_ops_noeq instead, the project keeps a second opclass in every later pg_trgm version, even after a cost-model fix makes it unneeded.

Message #3 says users can create the opfamily and opclass for themselves, as in patch 3, and rebuild their indexes. So shipping the opclass adds convenience and a documented name, not something that is impossible today. Message #6 argues that users should not have to hand-roll it, but neither message gives field reports or a count of affected users beyond the 2024 report that drew no replies. Users who need only LIKE or regex matching, and not the <-> ordering, can also use gin_trgm_ops, whose cost estimate message #1 says is already fixed.

**Question:** How many users hit this and cannot use GIN, and why is a documented user-defined opclass recipe not enough for them?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CA%2BvB%3DAHon6EB%3D90pbPDik_u8%2B%2B%2BTy8TA6Z4-RXo_X5QCr71kcQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/69c88bb3-d9a8-4054-9024-1f324d31e5aa%40dunslane.net), [#4](https://www.postgresql.org/message-id/CAPpHfdtGzomdU%2B0wiwrZS52NRQYwMrzDraE0eCfwAM2RvQAcag%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CA%2BvB%3DAG1P9FMm30ov_w4%2B%2ByNZ9Cdy40hYdC0tLxxPT2GEQaQxQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/ab8db543-bf0c-4a80-8d04-fe6eac817f3c%40dunslane.net); `already-exists-1`</sub>

## Questions for the author

1. Why does charging for a consistent call per index item in gistcostestimate, as message #4 suggests, not make btree win for = in the test case?
2. If GiST cost estimation for = is fixed later, how would gist_trgm_ops_noeq be retired without breaking dumps and pg_upgrade of existing indexes?
3. Which users hit this plan and cannot use gin_trgm_ops or a self-made opclass, for example because they need <-> ordering or lack superuser?
4. Why is a documented recipe for a user-defined opclass in the pg_trgm docs not enough?

## Touch points

- **catalog changes:** pg_trgm--1.6--1.7.sql adds opclass and operator entries for gist_trgm_ops_noeq.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Workaround opclass name becomes a permanent part of every dump
- **planner behaviour change:** The planner can no longer consider an index built with the new opclass for = conditions.
- **pg_dump / restore round-trip:** pg_dump writes gist_trgm_ops_noeq into CREATE INDEX, so restores need pg_trgm 1.7 or later.
- **backpatchability:** The fix is an extension version bump to 1.7, so released branches with the same plan problem do not get it.
- **maintenance burden ("who maintains this in 10 years"):** pg_trgm must ship and document two nearly identical GiST opclasses in every later version.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #3 says users can create the same opclass themselves with plain SQL today.

## Who would commit it, and when

- Owner: qualified committer engaged. Andrew Dunstan reviewed the patch and supports patch 3 as a stopgap in message #3; Alexander Korotkov, who takes part in the thread, prefers a fix to the cost estimate.
- Priority: needs design rethink; audience niche; review cost low. The patch is about 9 KiB of SQL, docs and tests in contrib/pg_trgm, and it affects only users with both a btree and a gist_trgm_ops index on one column. The thread must first choose between this opclass and a fix to the GiST cost estimate.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
