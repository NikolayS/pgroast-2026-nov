# Design pre-review: Parameterized append subpaths

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7339](https://commitfest.postgresql.org/patch/7339/); [thread](https://www.postgresql.org/message-id/flat/588d117fb042bbae3a6e7b4379ab8ec7%40postgrespro.ru), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at level 1: message #1 shows a hash join of about 1739 ms chosen over a 2.7 ms parameterized nested loop because a path was lost. Two fixable design points remain. maintenance-fit-1 applies only to the add_path() patch, which message #2 already moves away from. maintenance-fit-2 asks for the contract of the new FDW callback, and message #3 supports that design.

Not checked: The patch prose for the two patches in message #2 is not in the material, so the callback's contract and the cache's key and lifetime could not be checked. Whether the reparameterized path gets correct costs and quals needs the code.

## Prior attempts

- [A hook for path-removal decision on add_path](https://commitfest.postgresql.org/patch/2234/) (similar idea; Returned with feedback, 2021-10-05). reason not recorded Different now: P2 proposed an extension hook to decide path removal in add_path(). Patch 0001 changes the dominance rule in core, and the alternative design leaves add_path() alone and reparameterizes foreign paths through a new FDW callback.

## Objections a reviewer is likely to raise

### 1. Keeping all parameterized paths in add_path() costs every query

A user runs a 10-way join on local tables with many indexes and no foreign tables. Every base and join rel now keeps every differently parameterized path, so pathlists grow and join search does more work. The user sees higher planning time and memory after upgrade, with no plan change.

Patch 0001 changes add_path() in pathnode.c, which every rel of every query goes through. The problem shown in message #1 is specific to postgres_fdw partitions with use_remote_estimate, but this fix puts the cost on all planning. No planning-time or memory numbers are given. Message #2 already offers a narrower design, which suggests 0001 is not needed for the stated problem.

**Question:** What planning-time and memory change does 0001 cause on join-heavy queries with no foreign tables, such as a many-way join on indexed local tables?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/588d117fb042bbae3a6e7b4379ab8ec7%40postgrespro.ru), [#2](https://www.postgresql.org/message-id/98f84e7438c5b78a5a1c77539ffc7e0b%40postgrespro.ru); `maintenance-fit-1`</sub>

### 2. New FDW callback is a permanent API with no shown contract

An FDW author implements the callback and returns a path whose cost or restrictions do not match the new required_outer. reparameterize_path() trusts it, and the Append is built on wrong costs or wrong quals. Nothing in core checks the result, so the bug shows up as wrong plans or wrong results in the field.

Message #2 adds an FDW callback called from reparameterize_path(), but the material shows no name, signature, documentation, or rules on what the FDW must return or may refuse. Once released, FdwRoutine members are hard to change or remove. Message #3 points to an earlier proposal of the same API from the foreign-table-inheritance work, and no comparison is given. A committer would likely want one agreed contract before adding a member to FdwRoutine.

**Question:** What are the callback's signature and documented contract, and how does it differ from the earlier FDW API proposal mentioned in message #3?

<sub>fixable, level 2, ABI / extension API; messages [#2](https://www.postgresql.org/message-id/98f84e7438c5b78a5a1c77539ffc7e0b%40postgrespro.ru), [#3](https://www.postgresql.org/message-id/CAPmGK15wGDUj_gnzaS8JrL7xyDt19FGsfDs8nrSUii%2B1enKcCg%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Which design do you plan to pursue: the add_path() change from message #1 or the FDW reparameterization callback from message #2?
2. If the add_path() patch stays, what planning-time and memory change does it cause on a many-way join of indexed local tables with no foreign tables?
3. What are the callback's signature and documented contract, including when an FDW may decline to reparameterize a path?
4. How does the callback differ from the FDW API proposed in the foreign-table-inheritance work that message #3 points to?
5. What is the postgres_fdw cost cache keyed on, and is it freed at the end of each planning run?

## Touch points

- **ABI / extension API:** The alternative design adds a new FDW callback that reparameterize_path() calls.
- **planner behaviour change:** Both designs change which paths survive, so some queries get a parameterized nested loop where they got a hash join.
- **backpatchability:** A new FDW callback or a changed add_path() dominance rule limits the fix to master.
- **maintenance burden ("who maintains this in 10 years"):** A new FDW API member and a cost cache in postgres_fdw stay for good.
- **performance on the hot path:** Patch 0001 from message #1 changes add_path() in pathnode.c, which every query's planning goes through.

## Who would commit it, and when

- Owner: qualified committer engaged. Etsuro Fujita gave +1 to the callback design in message #3 and says he proposed this FDW API before. The add_path() patch would need someone who knows planner path dominance, and nobody has reviewed it.
- Priority: proceed; audience niche; review cost medium. The defect affects partitioned tables with postgres_fdw partitions and use_remote_estimate, but the shown plan is about 600 times slower than the lost one. The callback design keeps the cost on FDW users and has support in message #3. The patches from message #2 were not read, so review cost is an estimate.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
