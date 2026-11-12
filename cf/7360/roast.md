# Design pre-review: Skip LEFT/ANTI joins to a provably empty inner rel

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7360](https://commitfest.postgresql.org/patch/7360/); [thread](https://www.postgresql.org/message-id/flat/cba47b16-76cf-4d84-9dea-2421ccf83eee%40tantorlabs.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). No surviving objection is fatal, and none stops review at levels 1 or 2. Three fixable points remain: demand-1 asks for a realistic workload, standards-compat-1 asks whether the "stable" target means a back-patch, and maintenance-fit-3 asks for a comparison with a selectivity-only fix. A committer who knows outer-join Var handling has to judge whether the path substitution is safe. Confidence is low because only message #1 exists and no code was read.

Not checked: The code was not read, so this review could not check whether outer paths emit the join's reltarget correctly when the outer side is a join, an Append, or contains PlaceHolderVars. It also could not check test coverage for the all-partitions-pruned case.

## Prior attempts

None found before 2026-09-29.

## Objections a reviewer is likely to raise

### 1. No real-world query shown that hits an empty inner rel

A committer adds a new path-generation step to populate_joinrel_with_paths() and owns its interaction with nulling relids, partial paths and setrefs. If few production queries ever reach a LEFT or ANTI join whose inner rel is dummy and whose pushed-down quals are all IS NULL, the project keeps that code and its corner cases for little gain.

The only demonstration in message #1 uses a literal "AND false" in the ON clause, which a user can remove by hand. The partition-pruning case is named in one sentence but has no example, plan or field report. No -general or -bugs report, no user request and no other database are cited, so the size of the user group is unknown. The FULL JOIN attempt in message #1 failed with "wrong varnullingrels", which shows this area has real correctness risk to set against the gain.

**Question:** Which real workloads, such as ORM-generated queries or LEFT JOINs to partitioned tables pruned to nothing, produce this plan shape, and how often do users report the bad estimate?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/cba47b16-76cf-4d84-9dea-2421ccf83eee%40tantorlabs.com); `demand-1`</sub>

### 2. CommitFest target "stable" implies a plan change in minor releases

A user applies a minor release that carries this patch. Row estimates for LEFT and ANTI joins to empty rels change, so the plans for the joins above them change too. A query that was tuned or stable on the old plan picks a different join order or method, and the user finds out from a regression in production after the upgrade.

CommitFest entry 7360 sets the target version to "stable". The patch changes cardinality estimates and adds new path candidates in populate_joinrel_with_paths(), and the before/after plans in message #1 show a different plan shape. Back branches normally avoid planner changes that alter plans for existing queries without a bug that produces wrong results. Message #1 and the commit message frame the change as a performance improvement and do not ask for a back-patch, so the target may be wrong.

**Question:** Is this intended for master only, and if a back-patch is wanted, what makes this estimate change a bug fix rather than a plan change for existing queries?

<sub>fixable, level 1b, backpatchability; messages [#1](https://www.postgresql.org/message-id/cba47b16-76cf-4d84-9dea-2421ccf83eee%40tantorlabs.com); `standards-compat-1`</sub>

### 3. A narrower estimate fix may give most of the gain

The project adds a second path-level form of join removal, next to the existing left-join removal in analyzejoins.c. Later, a planner developer changes join removal or outer-join estimation and has two separate places to keep consistent. Users see different plans depending on which mechanism handled the join.

Message #1 names the row estimate as "the bigger issue". The cost of rescanning a one-time-false Result for each outer row is small. Making the selectivity of an "innervar IS NULL" qual equal to 1 when the inner rel is dummy would fix the estimate in the example without adding joinrel paths. The proposal does not compare against that option or say what the path substitution adds beyond it.

**Question:** How much of the gain in the example remains if only the IS NULL selectivity over a dummy inner rel is corrected and no outer paths are added?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/cba47b16-76cf-4d84-9dea-2421ccf83eee%40tantorlabs.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Is the patch meant for master only, and if a back-patch is wanted, what makes this estimate change a bug fix?
2. Which real workloads reach a LEFT or ANTI join with a dummy inner rel, for example a partitioned table with every partition pruned by the ON clause?
3. How much of the gain in the example remains if only the IS NULL selectivity over a dummy inner rel is corrected and no outer paths are added?
4. What is the planning-time cost of the new step for queries with many joins that do not qualify?
5. Were pushed-down quals that reference only outer Vars considered, and why does the patch not handle them?

## Touch points

- **planner behaviour change:** The patch changes row estimates and adds outer-rel paths for LEFT and ANTI joins to a dummy inner rel.
- **backpatchability:** The CommitFest entry sets the target version to "stable" for a change that alters plans.
- **maintenance burden ("who maintains this in 10 years"):** The new step in joinrels.c depends on nulling-relid rules that later outer-join work must keep in mind.
- **performance on the hot path:** The new check sits in populate_joinrel_with_paths(), and message #1 reports no planning-time numbers for queries with many joins.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the planner's join path generation and the nulling-relid rules for outer-join Vars. Nobody has replied to message #1 yet.
- Priority: proceed; audience unclear; review cost medium. The patch is about 11 KiB and changes one planner file plus tests, but its safety rests on nulling-relid rules that few reviewers know well. The only worked example uses a literal "AND false", so the number of affected users is unknown.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
