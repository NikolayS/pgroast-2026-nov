# Design pre-review: Improve row mark handling in self-join elimination

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7598](https://commitfest.postgresql.org/patch/7598/); [thread](https://www.postgresql.org/message-id/flat/CAHewXNn989Q4kP8Ewaeo8cBp-8jK6KX8CRs5UX6rAPMBtx54Pg%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). Only demand-1 survived, and it applies to 0002 alone. The thread shows no user query that self-joins one table with different lock strengths per alias, and message #7 does not answer Tom Lane's doubt in message #4 that the merging is worth the trouble. 0001 removes unreachable code and expands comments, and no objection to it survived. The verdict rests on missing evidence of demand, so confidence is low.

Not checked: Without the code, I could not check that the assertions in 0001 hold for every path into self-join elimination. I also could not check that the merge rule in 0002 matches a legal plan without self-join elimination for every pair of strength and wait policy.

## Prior attempts

- [Revert: Remove useless self-joins](https://www.postgresql.org/message-id/flat/2422119.1714691974@sss.pgh.pa.us) (related; Reverted, 2024-05-06). "We are quite late in the release cycle and new bugs continue to appear." Different now: P2 reverted the whole self-join elimination feature late in a release cycle because bugs kept appearing. This proposal is a narrow change to row mark handling in the re-committed feature, and 0001 only removes code.

## Objections a reviewer is likely to raise

### 1. No user is shown writing self-joins with differing lock strengths

A committer pushes 0002, which adds merge logic to the row-mark handling in analyzejoins.c, an area where SJE bugs have been found. Later fixes to that logic cost committer time and may need backpatching. The thread names no query that gains from the change, so the payoff for that cost is not known.

The only queries in the thread come from developers: message #1's example uses equal strengths, and its supposed bug was withdrawn in message #5. Tom Lane raised strength merging as a possible improvement in message #3, then wrote in message #4 that he was 'not really convinced that it's worth the trouble'. Message #7 does not answer that doubt; it ports his WIP code. The deadlock-on-upgrade argument is sound in principle, but no report shows a user who hit it.

**Question:** Which real applications or ORMs emit self-joins on a unique key with different FOR UPDATE/FOR SHARE strengths per alias, and has anyone reported the lock-upgrade deadlock?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/142952.1792872550%40sss.pgh.pa.us), [#4](https://www.postgresql.org/message-id/281282.1792889795%40sss.pgh.pa.us), [#5](https://www.postgresql.org/message-id/CAHewXN%3Dh3DmQ9Fc_aJN9dD%2BAbE90v%3Df4zmhSt_1tj1aVGTTzaQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAHewXN%3DofNonQSvzJ5a0csWtFJ%3DN5SAY_nJGPuNhkgKCs8svSg%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which real applications or ORMs emit self-joins on a unique key with different FOR UPDATE/FOR SHARE strengths per alias, and has anyone reported the lock-upgrade deadlock?
2. When the two aliases differ in both lock strength and wait policy, does the merged row mark always behave like some legal join order of the plan without self-join elimination?
3. Is 0002 meant for master only, and is 0001 meant to be committed alone if 0002 waits?
4. Which combinations of lock strength and wait policy do the new tests in join.sql cover?

## Touch points

- **planner behaviour change:** 0002 lets self-join elimination apply when two aliases carry locking row marks of different strength, so those plans change.
- **backpatchability:** The thread does not say whether 0002, a plan and locking behaviour change, targets master only.
- **maintenance burden ("who maintains this in 10 years"):** 0002 adds a merge rule for lock strength and wait policy to the row mark handling in analyzejoins.c.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane analyzed the case and wrote the WIP patch that 0002 builds on, but he has not reviewed v3 and doubted in message #4 that 0002 is worth it. The owner needs to know planner row marks and self-join elimination.
- Not now, show demand first: For 0002: a field report, a -general or -bugs thread, or an ORM-generated query that self-joins a table with different lock strengths per alias and hits the lock-upgrade deadlock or join-order dependence. A statement from Tom Lane that he now wants the merging would also change it. 0001 needs no such evidence.
- Priority: proceed; audience niche; review cost low. 0001 is a small cleanup in one file that follows the committer's own analysis and can go ahead alone. 0002 is also small but serves a query shape no user has reported, so it should wait for evidence of demand.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
