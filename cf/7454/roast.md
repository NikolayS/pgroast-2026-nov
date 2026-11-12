# Design pre-review: Avoid O(N^3) planning time in choose_plan_name()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7454](https://commitfest.postgresql.org/patch/7454/); [thread](https://www.postgresql.org/message-id/flat/179161574267.1707709.13345376006450096249%40gmail.com), 17 of 17 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). Nothing that survived questions the goal or the design. The patch fixes a planning-time regression from 8c49a484e8e, and v6 implements the threshold hash table that Robert Haas proposed. The one surviving objection, maintenance-fit-1, asks for a lasting regression test that crosses the 32-name threshold, which is a fixable point. A committer who knows the planner must still decide on v6 and on whether it goes into REL_19_STABLE.

Not checked: Without the code, nobody could check that the hash path and the list path choose the same name in every collision case, or that v6 never truncates its keys. The memory use of the hash table is also unmeasured.

## Prior attempts

None found before 2026-10-10.

## Objections a reviewer is likely to raise

### 1. Two lookup paths must agree, and only rare statements use the hash

A later patch adds a subplan kind or changes the naming rule, and updates the list search but not the hash entries. The regression suite has only small statements, so it stays green. A user whose generated SQL passes 32 names finds that their pg_plan_advice strings now point at different subplans.

v6 keeps the list and adds a simplehash above 32 names, so two structures must give the same name for every input (message #15). The author checked this with the threshold set to 0, but as a one-time run, and the prose does not say the suite keeps doing so. Names matter because advice strings refer to them (messages #12, #14). The list of touched files shows only subselect.sql, with no statement known to cross the threshold.

**Question:** Which regression test crosses the 32-name threshold with mixed generated and user-written names, so that a future change to one path alone fails the suite?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#12](https://www.postgresql.org/message-id/CA%2BTgmoZx7rAWBMMEjUnVhvqy8YsRarK8cfouw4DeiRHdgVbuxg%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CA%2BTgmoZ6GiKx_tFoT3RDR3TBFt3cPeOjonTB9mi29Sj1ZpK_7Q%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/179382931036.1743405.1568941311065599844%40gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which regression test crosses the 32-name threshold with mixed generated and user-written names, so that a later change to one lookup path alone fails the suite?
2. Is v6 meant for REL_19_STABLE before 19.0, or for master only?
3. How much planner memory does the hash table use for a statement with 4000 subplan names, compared with master?
4. Does the 32-name threshold hold for statements that mix generated and user-written names, beyond the two pure cases in measurements.txt section 3?
5. Should 19 keep the smallest-free-suffix naming, or switch to a single counter before release, given that pg_plan_advice strings refer to these names?

## Touch points

- **ABI / extension API:** The patch adds fields to PlannerGlobal in src/include/nodes/pathnodes.h, which extensions that include that header see.
- **planner behaviour change:** choose_plan_name() in planner.c changes how it finds a free subplan name; the author reports the chosen names stay the same.
- **backpatchability:** The regression is in 19 and no reviewer has said whether v6 goes into REL_19_STABLE before 19.0.
- **maintenance burden ("who maintains this in 10 years"):** The list search and the hash table must give the same name for every input from now on.
- **performance on the hot path:** Every statement with subplans calls choose_plan_name(); measurements.txt shows 2-4% slower small statements with default builds, equal with aligned builds.

## Who would commit it, and when

- Owner: qualified committer engaged. Robert Haas proposed the threshold hash table that v6 implements, and Tom Lane and Andres Freund commented on earlier versions. No one has reviewed v6 yet.
- Priority: proceed; audience niche; review cost low. Only generated SQL with many same-named subplans hits the problem, but it is a regression from v18 of up to about 1000 times in planning time. The patch is 14.2 KB, mostly in planner.c, and reuses the simplehash threshold pattern of ec_derives_hash.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
