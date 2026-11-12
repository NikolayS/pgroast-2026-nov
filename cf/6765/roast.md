# Design pre-review: teach vacuumlo to handle domains

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6765](https://commitfest.postgresql.org/patch/6765/); [thread](https://www.postgresql.org/message-id/flat/CALsgZNAM%3DAYK-9ZLR7Z0YLx6Lyx5aSrjjse3T%2BFmsJ%3DjTvfhDQ%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: message #1 reports field data loss with a repro, and the change makes vacuumlo delete less. The two surviving objections are fixable design points. maintenance-fit-1 asks to split the domain fix from composite and array support, and maintenance-fit-2 asks for a warning on type kinds vacuumlo still cannot search. Neither stops review at level 2.

Not checked: Without the code, I could not check that the generated expressions quote identifiers and cover nested cases correctly, or whether v2 adds tests. Messages #4 to #7 were not shown, so the scope of v2 rests on the brief.

## Prior attempts

None found before 2026-05-12.

## Objections a reviewer is likely to raise

### 1. Domain fix and composite/array expansion are bundled in one patch

A user on a released branch keeps a domain over oid. vacuumlo deletes those large objects, and v2 can only reach that user if a committer agrees to back-patch per-column delete expressions for composites and arrays. If no committer will take that on, the domain fix waits as well. Meanwhile the data loss continues on every supported branch.

v1 (message #4) was a small recursive query over typbasetype, which is a plausible back-patch candidate for a data loss bug. v2 (message #7) merges that fix with generated search expressions for composite and array types in one patch, 0001. Nobody has reviewed v2 on the visible thread across 4 CommitFests. The larger change is harder to review, test and back-patch, and few committers work on vacuumlo (message #2).

**Question:** Can the domain-over-oid fix be split out as a standalone first patch suitable for back-patching, with composite and array support in a separate patch?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/agPpPyLs-61qFw88%40nathan), [#4](https://www.postgresql.org/message-id/agU9JPim3XNwRJ4b%40nathan), [#7](https://www.postgresql.org/message-id/CAA5RZ0ufaaZmJP-JLHBz07QXhyfAGPY%2BMOsbk0-QQ3Mn8kH7Qw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Type kinds that are not covered are still skipped silently

A user stores large object OIDs in a range or multirange over oid, or in a container kind added in a later release. vacuumlo removes those objects without a message. The user finds out when lo_get() fails.

Message #5 says the approach can "cover all potential ways an OID or LO can be used". The brief lists only domains, composites and arrays, so ranges and multiranges appear to be left out. A future container kind would also be missed, and nothing would fail loudly when that happens. Message #1 asked for a warning about skipped columns, and the brief does not say v2 adds one. Without that warning, coverage depends on someone remembering to update vacuumlo.

**Question:** Does vacuumlo detect columns where oid or lo is nested in a type kind it cannot search, and warn about them, so that a missing case does not cause silent deletion?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CALsgZNAM%3DAYK-9ZLR7Z0YLx6Lyx5aSrjjse3T%2BFmsJ%3DjTvfhDQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAA5RZ0sF_p-ZE6LdVgzPr%3D9C32i2p1Pu7uxAPCV22-qh4tFh_Q%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Can the domain fix be a standalone first patch, with composite and array support in a second patch?
2. Does vacuumlo warn when a column nests oid or lo in a type kind it cannot search, such as a range or multirange?
3. Is the domain fix proposed for back-patching, given that the current behavior is documented?
4. Does the recursive catalog query run when a new vacuumlo connects to an older supported server?
5. How much longer does vacuumlo run on a database with many tables once it uses the deeper scan and the generated delete expressions?

## Touch points

- **security / privileges:** vacuumlo, often run as superuser, builds delete statements from type and attribute names read from the catalog.
- **backpatchability:** The patch fixes data loss on released branches, and the thread does not settle whether it is back-patched.
- **maintenance burden ("who maintains this in 10 years"):** vacuumlo gains a recursive catalog query and generated per-column search expressions that must follow new type kinds.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows contrib client tools and the type catalog. Nathan Bossart sent v1 and is listed as reviewer, but the material does not show his role, and no one has reviewed v2.
- Priority: proceed; audience niche; review cost medium. The patch fixes irreversible data loss for vacuumlo users who keep large object references in domain columns, and the cost falls only on vacuumlo. Review cost is medium because no code was read and v2 generates SQL; splitting out the domain fix would lower it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
