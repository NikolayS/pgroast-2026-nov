# Design pre-review: RLS bypass: ON CONFLICT DO UPDATE/SELECT evaluates WHERE before RLS check

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7403](https://commitfest.postgresql.org/patch/7403/); [thread](https://www.postgresql.org/message-id/flat/CABH8dKzYzRKD4izbushndHvvsEQg9b6m3w9ncYAGMbZ5m%2B9gNg%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the reorder itself; the leak has a repro and the new order matches the CREATE POLICY docs. The one surviving objection, standards-compat-1, is fixable and concerns the backpatch through 14, which turns a silent no-op into an RLS error in minor releases and is untested on 14 and 15. That is a per-branch risk decision for a committer who knows RLS and the executor.

Not checked: The code was not read, so it is unchecked whether the reorder covers every path where the WHERE clause sees the existing row, or changes locking or EvalPlanQual behavior. The DO SELECT part exists only on master, so the stable-branch patch would differ.

## Prior attempts

None found before 2026-10-06.

## Objections a reviewer is likely to raise

### 1. Backpatch turns silent no-op upserts into RLS errors in minor releases

A multi-tenant application runs `insert ... on conflict (id) do update ... where <false for other tenants>` under RLS. Today, a conflict with another tenant's hidden row is a silent no-op. After a 14.x–17.x minor update, the same statement raises the RLS error, and the application finds out in production through failed transactions.

Message #1 and the commit message state the behavior change and ask for Backpatch-through: 14. The proposal does not say whether any applications depend on a false WHERE skipping the error, and minor releases normally avoid such changes. The author tested only master and 16beta2, and DO SELECT exists only on master, so the backpatch covers just DO UPDATE on 14 and 15, untested there. A committer would likely ask whether the oracle needs this change on stable branches, or whether master-only plus a release note is enough.

**Question:** Which existing upsert patterns would start erroring after a minor update, and why does the leak justify that change on 14 through 17 rather than master only?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CABH8dKzYzRKD4izbushndHvvsEQg9b6m3w9ncYAGMbZ5m%2B9gNg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which existing upsert patterns start raising the RLS error after a minor update, and why does the leak justify that on stable branches instead of master only?
2. Does the leak reproduce on 14 and 15, and does the DO UPDATE part of the patch apply and pass the regression tests there?
3. Was the security team told before the public post, and is a coordinated release or CVE planned?
4. Do other paths, such as MERGE, evaluate user quals on an existing row before the RLS check?
5. What release-note text would warn users whose ON CONFLICT ... WHERE statements skip hidden rows today?

## Touch points

- **security / privileges:** The patch changes when the RLS conflict check runs relative to the ON CONFLICT WHERE clause, to close a leak of hidden column values.
- **backpatchability:** The commit message asks for Backpatch-through: 14, and a hidden conflicting row with a false WHERE now raises an error.
- **performance on the hot path:** The reorder is in ExecOnConflictUpdate() and ExecOnConflictSelect(), which run for every conflicting row of an upsert.

## Who would commit it, and when

- Owner: silent. Someone who knows row-level security enforcement and the ON CONFLICT code in nodeModifyTable.c, and who can judge behavior changes in stable branches. No committer has posted in the thread.
- Priority: needs champion; audience a sizeable minority; review cost low. The patch is 8.4 KB, touches one executor file plus rowsecurity tests, and fixes a leak for users who rely on RLS to hide rows. It has had no reply since 2026-10-06, and the backpatch decision needs a committer who knows the area.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
