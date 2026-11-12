# Design pre-review: pg_upgrade doesn't copy ICU rules on template0

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7568](https://commitfest.postgresql.org/patch/7568/); [thread](https://www.postgresql.org/message-id/flat/8c65df31-345e-429c-98fa-867f08d703b6%40iki.fi), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the bug is stated, a user reported it, and the fix completes what pg_upgrade already does for template0's other locale fields. The one surviving objection, maintenance-fit-1, is a fixable level 2 point about patch shape: v1 puts the daticurules fix and a rewrite of shared query and logging helpers in one commit that is requested for backpatch to 16. A committer has to decide what goes to stable branches.

Not checked: Without the code, this review could not check that the source-version checks still handle old clusters before 16, or that the StringInfo change keeps the helpers' behaviour for other callers. The CREATE DATABASE patch in message #7 was not read.

## Prior attempts

None found before 2026-10-28.

## Objections a reviewer is likely to raise

### 1. Back-branch fix bundles unrelated pg_upgrade refactoring

A committer backpatches v1 to 16 as one unit. It brings the StringInfo rewrite of the query and logging buffers in util.c and server.c, the errno change and the removal of the target-version branches. A regression in pg_log() or executeQueryOrDie() then reaches every pg_upgrade run on stable branches, not only clusters that use ICU rules.

The bug needs one more column in an existing UPDATE, but v1 also changes the shared logging and query helpers that every pg_upgrade step uses. The commit message lists all of it as one change, and message #5 asks for a backport to 16 without saying which parts go back. Heikki in message #1 suggested growing the buffers as a possible fix, but he expected ICU rules to stay short. Keeping the general refactoring on master only would limit what stable branches take on.

**Question:** Can the series be split into a minimal daticurules fix for 16 and later, plus the buffer and branch cleanup for master only, or does the fix depend on the buffer change?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/8c65df31-345e-429c-98fa-867f08d703b6%40iki.fi), [#4](https://www.postgresql.org/message-id/CAMPh8Mpz%3DVNv1kxDL%2BY3T4Ec_poGaFo3ZvVcEHq7-aHOQGt2kw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAGRkXqRE2%3DXsW9xzP4%3D3LEk9bWTwXWcTRoFuOAF935oTpfAGSw%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Can the series be split into a minimal daticurules fix for 16 and later, plus the buffer and branch cleanup for master only?
2. Does the daticurules fix depend on the StringInfo buffer change, and if so, at what rule length does the existing query buffer truncate?
3. Is there a test that upgrades a cluster initialized with multiline or long --icu-rules and checks template0's daticurules afterwards?
4. Should the CREATE DATABASE fix from message #7 go into this CommitFest entry or into a separate one with its own backpatch range?
5. Does the open question about why template0 keeps locale fields at all block this fix, or can the fix go in first?

## Touch points

- **on-disk format / pg_upgrade:** The patch changes what pg_upgrade writes into the new cluster's template0 row; no on-disk format changes.
- **catalog changes:** pg_upgrade now updates the existing pg_database.daticurules column for template0; no catalog schema change.
- **backpatchability:** The fix is requested for 16 and later, and v1 bundles a StringInfo rewrite of shared helpers in util.c and server.c.
- **maintenance burden ("who maintains this in 10 years"):** v1 changes the query and logging helpers that every pg_upgrade step uses, and removes the target-version branches.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas started the thread, is the registered reviewer and agreed the fix is right; the owner needs to know pg_upgrade and the ICU locale fields in pg_database.
- Priority: proceed; audience niche; review cost low. Only clusters initialized with --icu-rules are affected, but for them an upgrade silently drops template0's rules. The patch is about 10 KiB in five pg_upgrade files, a committer who knows the area is reviewing, and the open point is how to split it for backpatch.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
