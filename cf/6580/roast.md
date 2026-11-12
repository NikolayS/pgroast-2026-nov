# Design pre-review: Fix performance of REFRESH MATERIALIZED VIEW CONCURRENTLY

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6580](https://commitfest.postgresql.org/patch/6580/); [thread](https://www.postgresql.org/message-id/flat/40d694df-39fd-4a4a-9459-9d6489165f60%40gogi.tv), 22 of 22 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The slowdown in message #1 is reproducible, but the entry has no working design (prior-attempts-1). The author reports in message #20 that v7-0002 never skips a column, and the earlier unconditional skip gives wrong results for the (NULL, NULL) x2 case from message #16. Two level 1b questions also stand: a partial unique index already avoids the slow join (already-exists-1), and the only case is synthetic (demand-1). Confidence is medium because only 3 of 22 messages were read directly.

Not checked: The v7-0002 patch and messages #4 to #22 were not read directly; the no-op and the (NULL, NULL) x2 failure rest on the brief and the objections' account of messages #16 and #20. A new design would need a fresh review at level 2.

## Prior attempts

None found before 2026-02-09.

## Objections a reviewer is likely to raise

### 1. The only two designs tried are either incorrect or a no-op

A committer reviews entry 6580 as it is listed, Needs review, and commits v7-0002. Concurrent refresh on the reported matview stays about 3 times slower, because the guard never fires. If v7-0002 is relaxed back to the first version to recover the speed, the (NULL, NULL) x2 case is computed wrong again.

Message #20 reports that v7-0002 skips a column only when its index has a NOT NULL column. Matview columns are never marked NOT NULL, so the patch does nothing. The first version skipped nullable columns without a condition, and message #16 shows that version breaks the (NULL, NULL) x2 refresh. Message #20 proposes dropping 0002 and finding another approach, but entry 6580 still asks for review of the dead design.

**Question:** Which new design removes NULL-key rows from the diff join without matching (NULL, NULL) rows by mistake, and how does it avoid the NOT NULL flag?

<sub>fixable, level 2, design; messages [#6](https://www.postgresql.org/message-id/CAOVWO5qF2_FV7M%3DUSdSgjjPO124YMpw7TY%2BM4%2BmnkNPOA2Bstg%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/tencent_2DDB9D84961482D40AA55CC88CE0E51D740A%40qq.com), [#17](https://www.postgresql.org/message-id/CAOVWO5o46gqX_%3DBkrzuTRyPijDBWaYAGFVqn7UXrww_4owhJ0w%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAOVWO5pSxK9TZEEQ7xgNQOUFob_9F-u3YRRogXjDyRktgfaC%2BQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. A partial unique index already avoids the slow diff join

A user hits the slowdown from message #1 and waits for a core fix. They could write `create unique index on s(y) where y is not null` today: it enforces the same uniqueness, and refresh_by_match_merge() leaves it out of the join, so the refresh is fast again.

Concurrent refresh uses only unique indexes without a WHERE clause for the join condition, so a partial index takes no part in the diff join. Under the default NULLS DISTINCT, a unique index on y and one on y where y is not null accept the same rows. The thread offers adding NOT NULL as the remedy (message #6), but message #20 says matview columns cannot be NOT NULL, and v7-0002 changes nothing. A committer would likely ask whether a doc note covers the reported case with less risk than changing the join.

**Question:** Which users cannot replace the full unique index on the nullable column with a partial one, and what does a core change give them?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/40d694df-39fd-4a4a-9459-9d6489165f60%40gogi.tv), [#6](https://www.postgresql.org/message-id/CAOVWO5qF2_FV7M%3DUSdSgjjPO124YMpw7TY%2BM4%2BmnkNPOA2Bstg%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAOVWO5pSxK9TZEEQ7xgNQOUFob_9F-u3YRRogXjDyRktgfaC%2BQ%40mail.gmail.com); `already-exists-1`</sub>

### 3. Only a synthetic case from reading the code, with no field report

A committer redesigns the diff join in refresh_by_match_merge() for a pattern that may be rare: a matview with a unique index on a mostly-NULL column, refreshed concurrently. Every concurrent refresh then runs the new join logic. If that logic has a correctness bug, all such users get wrong matview contents to speed up a case few of them have.

Message #1 found the slowdown by reading matview.c and showing it with generate_series. Neither it nor any later message reports a production matview hit by it. Message #21 says the delete-and-reinsert path is costly only when there are many such rows. Dropping the extra unique index works around it, since concurrent refresh needs only one usable unique index.

**Question:** Which real deployments have unique indexes on mostly-NULL matview columns, and why can they not drop that index or refresh without CONCURRENTLY?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/40d694df-39fd-4a4a-9459-9d6489165f60%40gogi.tv), [#21](https://www.postgresql.org/message-id/CA%2BTgmoaryPLAMNWhF8cP4L%2Bf%2BwQKkwS5OGCC4rUCwfpm%2Bu9-fg%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which design keeps rows with NULL keys out of the delete-and-reinsert path without matching (NULL, NULL) rows by mistake, given that matview columns are never marked NOT NULL?
2. Should entry 6580 stay at Needs review, or be withdrawn until a new performance design is posted?
3. Which users cannot replace the full unique index on the nullable column with a partial index `where y is not null`, and what does a core change give them?
4. Which real deployments refresh a matview concurrently with a unique index on a mostly-NULL column?
5. What does the new design cost a concurrent refresh when the nullable unique-index columns hold no NULLs?

## Touch points

- **backpatchability:** The report is on pgsql-bugs, and the thread does not say whether the fix targets back branches or only version 20.
- **maintenance burden ("who maintains this in 10 years"):** The patch changes the diff join in refresh_by_match_merge(), where a mistake gives wrong matview contents, and adds no test.
- **performance on the hot path:** Every concurrent refresh runs the changed join, and the author expects a small slowdown when nullable index columns hold no NULLs.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A partial unique index `where y is not null` already keeps the column out of the diff join.

## Who would commit it, and when

- Owner: silent. The owner would need to know the diff logic in refresh_by_match_merge() and record equality semantics. Robert Haas took part in the thread, but only on the duplicate-row fix (CF 6579).
- Priority: needs design rethink; audience niche; review cost medium. The slowdown needs a unique index on a mostly-NULL matview column, and a partial index avoids it today. The change is confined to matview.c with no user-visible surface, but the current patch does nothing; review cost is a guess because the performance patch was not read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
