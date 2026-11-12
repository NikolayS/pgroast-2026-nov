# Design pre-review: Fix exclude generated columns in COPY TO with RLS

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7589](https://commitfest.postgresql.org/patch/7589/); [thread](https://www.postgresql.org/message-id/flat/1915ae77-392a-4bdf-b3e2-157d63af320c%40data-bene.io), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal or the design. The patch makes COPY TO under RLS follow the documented rule and the non-RLS path. The one surviving objection, standards-compat-1, asks which branches the fix is for, because a back-patch would change the output columns in a minor release. That is a fixable point for a committer to decide, and it does not stop review of the idea.

Not checked: The code was not read. This review could not check that policy quals on generated columns still apply, that virtual generated columns are excluded, or that partitioned and inherited tables behave the same.

## Prior attempts

None found before 2026-11-04.

## Objections a reviewer is likely to raise

### 1. Backpatch would change COPY TO output columns in a minor release

An export job runs COPY t TO under an RLS-restricted role and its consumer expects the generated column in a fixed column position. After a minor update the column disappears and every later field shifts by one. The consumer then loads wrong values or rejects the rows, and the error shows up downstream long after the upgrade.

The CommitFest entry is tagged Bugfix, and its target version is None. The proposal does not say whether the fix goes to back branches or only to master. Since generated columns arrived, the RLS path has emitted them, so some exports may depend on today's column layout. Matching the documentation is a good reason to change master, but changing output shape in a minor release is a separate decision that the thread has not discussed.

**Question:** Is this fix meant for master only, and if it is meant for back branches, how would users who rely on the current RLS output learn about the change?

<sub>fixable, level 1b, backpatchability; messages [#1](https://www.postgresql.org/message-id/1915ae77-392a-4bdf-b3e2-157d63af320c%40data-bene.io), [#3](https://www.postgresql.org/message-id/1936f1d8-e530-4d03-8af7-c65552153cb3%40data-bene.io); `standards-compat-1`</sub>

## Questions for the author

1. Is the fix meant for master only, and if for back branches, how would users who rely on the current RLS output learn about the change?
2. Does an RLS policy whose expression references a generated column still apply correctly when the internal SELECT target list omits that column?
3. How does the RLS path treat virtual generated columns after the patch, and does a regression test cover them?
4. Does the new target list behave the same for partitioned tables and inheritance parents on the RLS path?

## Touch points

- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Backpatch would change COPY TO output columns in a minor release
- **security / privileges:** The patch changes which columns the RLS rewrite of COPY TO reads, so the SELECT privilege check no longer covers generated columns.
- **backpatchability:** The entry is tagged Bugfix with target version None, and a back-patch would change COPY TO output columns under RLS.

## Who would commit it, and when

- Owner: silent. A committer who knows the COPY code and the RLS rewrite in DoCopy() would have to own it. The visible thread has the author and one reviewer.
- Priority: proceed; audience niche; review cost low. The bug affects only COPY TO without a column list on a table with RLS and a generated column. The patch is about 6 KiB in copy.c plus rowsecurity tests, and it reuses the CopyGetAttnums() list.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
