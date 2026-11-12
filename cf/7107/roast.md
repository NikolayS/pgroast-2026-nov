# Design pre-review: BUG #19603: NaN coordinates in cube type break comparison and GiST scans

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7107](https://commitfest.postgresql.org/patch/7107/); [thread](https://www.postgresql.org/message-id/flat/19603-7b1f783d5bbfe791%40postgresql.org), 9 of 9 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the NaN semantics, which follow float8 practice. The two surviving objections, standards-compat-1 and replication-ondisk-1, are fixable rollout points. The patch changes the comparisons behind the cube btree operator class, the thread covers REINDEX only for GiST, and the entry targets stable. A committer has to decide between back branches and master only.

Not checked: Only the v1 commit message and 3 of 9 messages were visible, so the scope of v4, the GiST -Infinity lower bound logic and the cost of the added NaN checks could not be judged.

## Prior attempts

- [Fix behavior of geo_ops when NaN is involved](https://www.postgresql.org/message-id/flat/CAGf+fX70rWFOk5cd00uMfa__0yP+vtQg5ck7c2Onb-Yczp0URA@mail.gmail.com) (similar idea; Returned with feedback, 2022-08-01). "Rather than code changes, what this patch needs is more community interest." ([message](https://www.postgresql.org/message-id/f84318e2-2918-8c21-436f-a79823c9e097%40timescale.com)) Different now: P1 made NaN behave consistently in core geo_ops. This proposal does the same for contrib/cube, starts from a bug report, and has an engaged reviewer.

## Objections a reviewer is likely to raise

### 1. Back-patching changes cube btree and GiST ordering in minor releases

A user has a btree index or unique constraint on a cube column that holds NaN coordinates. After a minor upgrade, the new comparison function no longer matches the order the existing index was built in. Index scans then miss rows, or a unique index accepts values it used to reject, and the user sees no error.

The CommitFest entry targets "stable", but the patch changes `=`, `<` and `>` for cube, and these operators back the cube btree operator class. Message #5 covers REINDEX only for GiST, and no message in the thread mentions btree indexes. Before the patch '(1,1)' = '(nan,nan)' returned true (message #3), so a unique index built on the old behaviour can hold values that are inconsistent under the new behaviour. A committer would likely ask whether a silent change in index semantics belongs in a minor release, even as a bug fix.

**Question:** Which index types on cube columns become invalid after this change, and why is a minor release the right place for that change rather than master only?

<sub>fixable, level 2, backpatchability; messages [#3](https://www.postgresql.org/message-id/CAJ7c6TM7CNrQShuPapof77A6SoP-qqjpQX2q1C3B0HT4749T6A%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAJTYsWWQhEA_FS8%3DeKbZ%2BcmuAHhHXBv%3DPaGQjExZu0VKQSm6yw%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Btree indexes and unique constraints on cube need reindex too

A user has a btree index or a unique constraint on a cube column holding NaN coordinates, built with the old cube_cmp order. After the update, the new comparison order no longer matches the order of the index pages. Lookups miss rows and the unique check can admit duplicates, and nothing reports it unless the user runs amcheck.

Message #5 changes comparisons to the float8 helpers, so NaN now sorts above all finite values. Message #3 shows that the old code treated '(1,1)' and '(nan,nan)' as equal. Each btree index on cube built under the old order is therefore out of order for NaN keys. Message #5 asks for a release note about REINDEX only for GiST, and the brief notes that btree is not addressed.

**Question:** Which index types and constraints on cube columns must be rebuilt after the fix, and how can a user find the affected ones before updating?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#3](https://www.postgresql.org/message-id/CAJ7c6TM7CNrQShuPapof77A6SoP-qqjpQX2q1C3B0HT4749T6A%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAJTYsWWQhEA_FS8%3DeKbZ%2BcmuAHhHXBv%3DPaGQjExZu0VKQSm6yw%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Which index types and constraints on cube columns (btree, unique, GiST) must be rebuilt after the fix?
2. Is the comparison and ordering change meant for back branches, and if so, why is a REINDEX requirement in a minor release better than a master-only fix?
3. Could the distance fix go to back branches while the comparison and ordering change goes to master only?
4. How can a user find the cube indexes that hold NaN coordinates before updating?
5. Was rejecting NaN in cube input considered as an alternative, and why was it not chosen?

## Touch points

- **on-disk format / pg_upgrade:** Existing GiST and btree indexes on cube columns with NaN no longer match the new comparison and distance rules; pg_upgrade keeps index files.
- **planner behaviour change:** see objection standards-compat-1: Back-patching changes cube btree and GiST ordering in minor releases
- **backpatchability:** The CommitFest entry targets stable, and the fix changes cube =, < and > and GiST distance results that existing indexes depend on.
- **performance on the hot path:** Cube comparisons and GiST distance calls gain NaN handling through the float8 helpers; no measurement is shown.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows GiST opclass support functions and back-patch policy for changes to index ordering. The visible thread shows a reviewer but no committer.
- Priority: proceed; audience niche; review cost medium. Only users with NaN coordinates in cube values benefit, but the current results are wrong and the change stays inside contrib/cube with existing float8 helpers. Review cost is medium because the GiST key logic and the back-branch decision need care, and v4 was not read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
