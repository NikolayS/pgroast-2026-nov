# Design pre-review: [BUG] Fix EXCLUDED virtual generated columns using tableoid

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7295](https://commitfest.postgresql.org/patch/7295/); [thread](https://www.postgresql.org/message-id/flat/CAHg%2BQDf7wTLz_vqb1wi1EJ_4Uh%2BVxm75%2Bb4c-Ky%3D6P%2ByOAHjbQ%40mail.gmail.com), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the choice to raise an error: v2 follows Dean Rasheed's argument that EXCLUDED has no system columns. Two fixable design points remain. standards-compat-1 asks whether a back-patch to v18 may make statements fail that run today. standards-compat-2 asks whether stored and virtual columns over tableoid now behave differently in EXCLUDED. Both are release and consistency decisions for a committer who knows this area.

Not checked: The v2 patch and its commit message were not shown, so the placement of the check and its coverage of all expansion paths could not be judged. The behaviour of the unpatched WHERE cases rests on the objection's account of message #11.

## Prior attempts

- [COPY WHERE clause generated/system column reference](https://www.postgresql.org/message-id/flat/CACJufxHb8YPQ095R_pYDr77W9XKNaXg5Rzy-WP525mkq+hRM3g@mail.gmail.com) (related; Withdrawn, 2026-01-08). "This seems fundamentally wrong, because generated columns are a property of the target table, but the COPY WHERE clause is meant to filter the input data." ([message](https://www.postgresql.org/message-id/70f3402f-30a2-48e1-9fb3-a4da2e77296f%40eisentraut.org)) Different now: P1 tried to make generated columns work in COPY FROM WHERE, which filters input rows. v2 here does what that thread chose: it rejects the reference with a clear error instead of giving the proposed row a value.

## Objections a reviewer is likely to raise

### 1. Back-patching a new error breaks statements that run today on v18

An application on 18.x runs INSERT ... ON CONFLICT DO UPDATE ... WHERE excluded.c IS NOT NULL, where c is a virtual column over tableoid. Message #11 shows this succeeds and updates rows on the unpatched build. After a minor update with v2, the same statement fails with the new error, and the application stops writing at upgrade time.

Message #11 shows WHERE cases that run without error before v2 and fail after it, on plain and partitioned tables. The CF entry targets stable, and message #10 leaves the back-patch open. Turning an internal error into a user-facing error is safe to back-patch, but these cases do not raise an internal error today. The proposal does not say why they work or what tableoid value they see, so nobody can judge whether their results are wrong enough to justify failing them in a minor release.

**Question:** Do the WHERE cases that succeed today give wrong results, and if not, why should a minor release make them fail?

<sub>fixable, level 2, backpatchability; messages [#5](https://www.postgresql.org/message-id/18fbeb33-9631-4f6e-9280-2c787aef254e%40gmail.com), [#10](https://www.postgresql.org/message-id/CAJTYsWXH1WxRbD3KBrxMm%2BApHKC_7zwhhY2BhK9Gk1fOZHKo9A%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CADMs5OPKxkM53-R_Us5MG0NHoAkQrjckcPw3dRLj4sPaTa7_%2BA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Stored and virtual columns over tableoid may differ in EXCLUDED

A user defines c as GENERATED ALWAYS AS (tableoid) STORED and uses excluded.c in ON CONFLICT DO UPDATE. They then change c to VIRTUAL. If the stored form likely yields the target partition's oid, the same statement now fails with the new error, and the user finds out only when it runs.

Dean's case in message #9 compares the rule with a direct reference to excluded.tableoid, which is a reasonable comparison. It does not cover a stored column over tableoid, whose value is computed for the proposed row before the conflict check. Message #11 tested a stored column over b * 10 only. If the stored form works, v2 adds a new difference between virtual and stored columns, and the documentation would need to describe it.

**Question:** What does excluded.c return for a stored generated column over tableoid, and is the virtual-only error documented as an intended difference?

<sub>fixable, level 2, design; messages [#8](https://www.postgresql.org/message-id/CAJTYsWW2uGkcL%3DE7GuOWTtzFSxh3wEm%2BxGz63XquhOJ0kywr5g%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAEZATCUQaAuBiKyHrUevTWsnM9yxSNtgAKp2twyv_eQ2mHkYpw%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CADMs5OPKxkM53-R_Us5MG0NHoAkQrjckcPw3dRLj4sPaTa7_%2BA%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Do the WHERE cases that run without error on unpatched v18 return wrong results, and which tableoid value do they see?
2. If those WHERE cases return correct results, why should a minor release of v18 make them fail?
3. What does excluded.c return for a stored generated column whose expression uses tableoid?
4. If the stored form works, is the virtual-only error an intended difference, and where will the documentation describe it?
5. Will the patch go to v18 as well as master, and does the v2 commit message say so?

## Touch points

- **SQL-standard conflict or non-standard syntax:** v2 makes ON CONFLICT DO UPDATE reject excluded.<virtual column> when the column's expression uses tableoid.
- **planner behaviour change:** The check sits in expand_virtual_generated_columns() in prepjointree.c and replaces a planner internal error.
- **backpatchability:** The CF entry targets stable, and v2 adds an error for WHERE cases that run today on v18.
- **maintenance burden ("who maintains this in 10 years"):** A new user-facing error message and a virtual-only rule for tableoid in EXCLUDED need to stay consistent with stored columns.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut is the assigned committer on the CF entry, and Dean Rasheed set the direction in the thread. The owner needs to know virtual generated column expansion and ON CONFLICT planning.
- Priority: proceed; audience niche; review cost low. Few users define a virtual generated column over tableoid and reference it through EXCLUDED, but the current result is an internal error in v18. The check is small, a committer is assigned, and the open points are the back-patch decision and the stored-column comparison.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
