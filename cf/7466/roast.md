# Design pre-review: Fix WITHOUT OVERLAPS PKs used for functional grouping

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7466](https://commitfest.postgresql.org/patch/7466/); [thread](https://www.postgresql.org/message-id/flat/CA%2BrenyWoOf6dM_%2Be6Zath%2B0i5Lr%2Buzn1hz7DAuSaV_A5Tz6XhA%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The patch fixes a wrong-results bug that a committer reported, and the reporter backed the narrow fix in message #4. One fixable design point survived, replication-ondisk-1: views stored on 18.x that rely on a WITHOUT OVERLAPS key are not re-checked and fail later at restore or pg_upgrade. That is a backpatch decision for a committer who knows the area, and it does not question the fix itself.

Not checked: The code was not read, so the claim that the check accepts only btree indexes and leaves ordinary primary keys alone is unverified. Whether stored views are in fact not re-checked after the update rests on the surviving objection's description.

## Prior attempts

- [Revert "Add GROUP BY ALL".](https://www.postgresql.org/message-id/flat/5243308F-8E5C-45AA-828C-FAD96C4F34DA@gmail.com) (related; Reverted, 2026-07-17). "This caused the query to return wrong results when ORDER BY specifies non-default equality semantics." Different now: P11 reverted a new grouping feature that ignored non-default equality semantics. This patch removes an existing inference with the same class of flaw and also takes the narrow route.

## Objections a reviewer is likely to raise

### 1. Stored views that use the temporal key break later dump and upgrade

On 18.0, a user creates a view that groups by a WITHOUT OVERLAPS primary key and selects other columns. After the minor update, the stored view still runs and can still return wrong results for citext. The failure appears later, when pg_dump restore or pg_upgrade to 19 recreates the view and hits the new error partway through.

Functional grouping is checked only at parse analysis, and the view records a pg_depend entry on the primary key constraint, so existing views are not re-validated. The commit message and message #1 cover new queries only, and the brief notes nothing about release notes. With Backpatch-through: 18, this catalog state is already in released clusters. Message #4 backs the simpler fix, but no message covers objects that already exist.

**Question:** How will users on 18.x find views and rules that already depend on a WITHOUT OVERLAPS key for functional grouping, before a dump or pg_upgrade fails on them?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#1](https://www.postgresql.org/message-id/CA%2BrenyWoOf6dM_%2Be6Zath%2B0i5Lr%2Buzn1hz7DAuSaV_A5Tz6XhA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/nwsx2isaffevptmws7anxv2xsc66rlmarom6a2bvc6jug3hsqh%40mtwkrt5r3psf); `replication-ondisk-1`</sub>

## Questions for the author

1. How will users on 18.x find views and rules that already depend on a WITHOUT OVERLAPS key for functional grouping, before a dump or pg_upgrade fails on them?
2. Will the release notes warn that queries grouping by a WITHOUT OVERLAPS primary key can start to fail after the minor update, and say how to rewrite them?
3. Should existing views that rely on a WITHOUT OVERLAPS key keep running after the minor update, given that they can still return wrong results for citext?
4. Does a citext column that gets case-sensitive GiST comparisons through gist_text_ops need a separate fix outside WITHOUT OVERLAPS?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade re-creates stored views that group by a WITHOUT OVERLAPS key, and the new error can stop it.
- **catalog changes:** The patch changes check_functional_grouping in pg_constraint.c; existing pg_depend entries from views on the temporal key stay.
- **SQL-standard conflict or non-standard syntax:** The set of queries accepted under functional dependency for GROUP BY becomes smaller.
- **planner behaviour change:** A WITHOUT OVERLAPS primary key no longer proves functional dependency, so some queries now fail at parse analysis.
- **pg_dump / restore round-trip:** A dump of a view created on 18.x that relies on the temporal key fails to restore on a patched server.
- **backpatchability:** The commit message says Backpatch-through: 18, and working queries start to fail after a minor update.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund reported the bug and backed the simpler fix in the thread; Peter Eisentraut is the listed reviewer but has not posted. The owner needs to know parse analysis of GROUP BY and the temporal key code.
- Priority: proceed; audience niche; review cost low. This is a wrong-results fix for users of temporal primary keys, in one backend file plus regression tests. The open point is how the backpatch treats views already stored on 18.x, which a release note and a catalog query can cover.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
