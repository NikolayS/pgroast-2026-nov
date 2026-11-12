# Design pre-review: Implement missing join selectivity estimation for range types

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6668](https://commitfest.postgresql.org/patch/6668/); [thread](https://www.postgresql.org/message-id/flat/CAB4o4asMq3k6HN9WfDsssQ5DDVfAziB4TpiFJ8RBJgZTVuwC7g%40mail.gmail.com), 32 of 32 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal or the design of the estimator. demand-1 shows the thread has no user report, but the cost falls only on queries that join on range columns, so it is a request for evidence. maintenance-fit-2 is decisive: no committer has reviewed the v5 to v9 rework, and the January 2024 ASan crash is not shown to be gone.

Not checked: The code was not read, so the ASan crash from January 2024, the bounds handling in the merge walk and the coverage of the security check in every estimator path could not be judged.

## Prior attempts

None found before 2022-06-30.

## Objections a reviewer is likely to raise

### 1. No user report shows range or multirange joins being misestimated

A committer takes on a new histogram merge-walk estimator for range and multirange joins, plus a new privilege-check path, and owns its bugs and plan changes. If few production queries join on <<, >> or && between range columns, the plan gains reach few users while the ASan history and security surface stay with the project.

The patch began as a course project (message #7), and Tomas Vondra said he did not remember ever seeing an inequality join on a range column (message #6). The demand signals are reviewers calling it useful (message #15) and synthetic tables built to show the effect (messages #7, #20). No -general or -bugs report, production plan or user workload appears in the thread. Tom Lane's point (message #13) that scalar priority is no reason to reject is a different question from whether range joins have users.

**Question:** Which real workloads, such as temporal or scheduling schemas, join on range <<, >> or && today, and what bad plans do they get from the 0.005 default?

<sub>fixable, level 1b, demand; messages [#6](https://www.postgresql.org/message-id/d46e1950-9228-3fe2-02b6-72ab7aaafb3a%40enterprisedb.com), [#7](https://www.postgresql.org/message-id/CAB4o4asvPN%3DNT7KvS9zVQjZbdsiRW5t8aQctEkW7mxc4hbBxVQ%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/2516071.1699991181%40sss.pgh.pa.us), [#15](https://www.postgresql.org/message-id/5fc8d7f6-9cd5-4da3-8a94-1d06831441fe%40yandex.ru), [#20](https://www.postgresql.org/message-id/CABXr29HUT2m7YPpB4gOZHphYmwpwuEWFZ_p6oeyouhWw8Zc1Tw%40mail.gmail.com); `demand-1`</sub>

### 2. No committer has engaged with the 2026 series

A committer pushes v9 late in a cycle. A buildfarm member later reproduces the ASan out-of-bounds read reported in messages #16 and #18, or a user reports a plan regression on range joins. The committer must then fix an estimator that no other committer has reviewed since 2023.

The 2023 committer comments (messages #2-#6, #13) were about v1-v3. The v5-v9 rework was reviewed only by co-authors and non-committers (messages #21-#32), and message #28 says a committer is still needed. The original code came from a course project (message #7), and message #17 says the author could not confirm the CFbot crash was fixed. The CF entry has crossed three CommitFests and now needs a rebase.

**Question:** Has a committer familiar with selfuncs reviewed the v9 design, and is the 2024 ASan failure shown to be gone on CFbot?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#7](https://www.postgresql.org/message-id/CAB4o4asvPN%3DNT7KvS9zVQjZbdsiRW5t8aQctEkW7mxc4hbBxVQ%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/2516071.1699991181%40sss.pgh.pa.us), [#16](https://www.postgresql.org/message-id/CALDaNm29NMhpPvoM49qqfYzVcRJq0CPTd-xSUcTF7RZQO2TASQ%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/08b2dea8-6dca-06e4-a53d-3e6334e7645b%40ulb.be), [#18](https://www.postgresql.org/message-id/CALDaNm3A4j7%3DrmY128Eb3v8zi%3DhHAnGizUcbSX5ekZKHy-wmMA%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CABXr29Edby6Cbbaps_OQh-%2BkvKx%2BXFNe26jc%3DsBxWhjaYO9raQ%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/6929b807-32e9-4e80-b3d3-1f7823880d60%40gmail.com), [#32](https://www.postgresql.org/message-id/CABXr29GKcW58pJ%2BmqZNXjAjLEmGdsDY%2BH-5bmrkgiDywjWnBJQ%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Is the ASan out-of-bounds read in calc_hist_join_selectivity, reported in messages #16 and #18, shown to be gone in v9 on CFbot?
2. Which real workloads join on range <<, >> or && today, and what bad plans do they get from the 0.005 default?
3. What is the estimation error of v9 for <<, >> and && only, including data with many duplicate ranges?
4. How much planning time does the merge walk add for a range join at a statistics target of 10000?

## Touch points

- **catalog changes:** The patch adds join selectivity functions to pg_proc.dat and sets them as oprjoin in pg_operator.dat.
- **planner behaviour change:** Row estimates and plans change for joins on range and multirange <<, >> and &&.
- **security / privileges:** The estimators run comparator and subtype_diff functions over statistics, guarded by statistic_proc_security_check() since v9.
- **maintenance burden ("who maintains this in 10 years"):** Two new histogram-based join estimators and a shared private header stay in the planner's selectivity code.
- **performance on the hot path:** The merge walk runs at plan time for range join clauses; its cost at high statistics targets is not shown.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the selectivity estimation code in selfuncs. Tomas Vondra and Tom Lane commented on v1 to v3 in 2023, and no committer has replied on v5 to v9.
- Priority: needs champion; audience niche; review cost medium. The goal is valid and costs fall only on range joins, but message #6 says such joins are rare and no committer has reviewed v5 to v9. Review cost is judged from the touched files; the code was not read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
