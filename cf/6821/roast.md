# Design pre-review: Unify parallel worker handling for index builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6821](https://commitfest.postgresql.org/patch/6821/); [thread](https://www.postgresql.org/message-id/flat/CAP53Pkw3WcVbg61G29AmN%3D6XkuxGryoWKnhTyC1JmkT08i8qVA%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived the critic. The only objection raised, maintenance-fit-1, was refuted because Postgres does not keep its C API stable across major versions. The series fixes two reported bugs and removes duplicated code, with support in messages #2 and #3. Confidence is low because nobody has reviewed v2, and the wrong-results claim has no reproducing test in the material.

Not checked: The code was not read, so the wrong-results claim, the safety of the 0002 backpatch for extension ABI, and the behaviour of ParallelIndexBuildScan() in 0005 are unchecked. Nobody has reviewed v2 yet.

## Prior attempts

None found before 2026-05-31.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is there a test or script that reproduces wrong results from a parallel BRIN or GIN build that met a broken HOT chain?
2. Which patches go to the back branches, and do the 17 and 18 versions of 0002 differ from the master version?
3. Is the GIN query ID fix in 0001 also meant for backpatch to 18?
4. How does an out-of-core AM learn that ParallelIndexBuildEnd now requires workers to report WAL and buffer usage?
5. Can an AM keep its own fields under the shared mutex, or must it add a second lock as the pgvector sketch does?

## Touch points

- **catalog changes:** Patch 0002 makes parallel BRIN and GIN builds set the existing pg_index.indcheckxmin flag; no catalog definition changes.
- **ABI / extension API:** New header access/parallel_index_build.h and query text helpers in access/parallel.h are usable by out-of-core index AMs.
- **planner behaviour change:** With indcheckxmin set, transactions with older snapshots no longer use a newly built BRIN or GIN index.
- **backpatchability:** Patch 0002 proposes a backpatch to 17 for BRIN and 18 for GIN.
- **maintenance burden ("who maintains this in 10 years"):** A new shared module in access/common becomes the path for parallel builds in nbtree, GIN and BRIN.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier supported the query text and index build helpers in message #3, but only for v1. The backpatched fix in 0002 needs someone who knows index builds, HOT chains and indcheckxmin.
- Priority: proceed; audience a sizeable minority; review cost medium. Patch 0002 fixes a reported wrong-results case in parallel BRIN and GIN builds, and the refactoring removes duplicated code without new user-facing surface. Review cost is medium because 0003 and 0005 are large and touch three index AMs; 0001 and 0002 are small and can go first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
