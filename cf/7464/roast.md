# Design pre-review: WITHOUT OVERLAPS foreign key allows referencing EXCLUDE constraint

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7464](https://commitfest.postgresql.org/patch/7464/); [thread](https://www.postgresql.org/message-id/flat/CA%2BrenyUy%3D%2BoZoOg0qN_wAHTL3MwVEnp4zdfX%2BBu54foz6OXpQw%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The fix closes a correctness hole reported by a committer, and no objection survived at levels 1a or 1b. Two fixable level 2 points remain about back-branch behaviour: standards-compat-1 (a dump from an earlier 18.x fails to restore the foreign key on a fixed 18.x) and standards-compat-2 (the 18-only warning assumes the fix lands before 19.0). Both are release-policy choices that a committer who knows back-patching practice has to settle.

Not checked: The code was not read, so the new error texts, the pg_upgrade report format and the exact conditions and frequency of the 18.x warning could not be checked. The material also does not show whether 19.0 has shipped, which decides whether standards-compat-2 applies.

## Prior attempts

- [Revert temporal primary keys and foreign keys](https://www.postgresql.org/message-id/flat/d0b64a7a-dfe4-4b84-a906-c7dedfa40a3e@eisentraut.org) (related; Reverted, 2024-05-16). "This feature set did not handle empty ranges correctly, and it's now too late for PostgreSQL 17 to fix it." Different now: P1 reverted temporal keys because they mishandled empty ranges. This patch closes a remaining gap of that kind: a temporal foreign key could reference a plain EXCLUDE constraint, which does not reject empty ranges.

## Objections a reviewer is likely to raise

### 1. Back-patched error breaks dump and restore within the 18.x series

A user on 18.1 has a temporal foreign key on a plain exclusion constraint and dumps the database. They restore it into 18.3, which has the fix. pg_restore reports an error for that constraint and continues, so the table loads without the foreign key, and the user may notice only if they read the error output.

Patch 0001 is marked Backpatch-through: 18, and its commit message says restoring such keys fails. The usual expectation is that a dump taken on a minor release restores on a later minor release of the same branch. Patch 0002 adds a warning on 18, but the warning appears only on the source server and does not prevent the restore failure. A committer would likely ask whether 18 should reject only new definitions and keep accepting existing ones, for example during restore, while 19 and later reject them outright.

**Question:** Why should 18.x reject these definitions at restore time instead of warning, given that a dump from an earlier 18.x release then loses the foreign key?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#1](https://www.postgresql.org/message-id/CA%2BrenyUy%3D%2BoZoOg0qN_wAHTL3MwVEnp4zdfX%2BBu54foz6OXpQw%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Warning code is limited to 18, but 19.0 may already allow these keys

If 19.0 ships before the fix, a user on 19.0 can create such a key. After a minor update to 19.1, the server neither warns nor checks. The key keeps allowing serialization anomalies, and the user finds out only at the next dump or major upgrade.

Message #1 limits the warning to 18 because the fix is expected to land before 19 is released. The CommitFest entry targets 18, it is still in Needs review, and the review date is after the usual major release window. If 19.0 has shipped, the reasoning behind the 18-only split in patch 0002 no longer holds, and the 19 branch would need the warning as well.

**Question:** If 19.0 ships without the fix, will patch 0002 also go to REL_19_STABLE, or what else warns users with keys created on 19.0?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CA%2BrenyUy%3D%2BoZoOg0qN_wAHTL3MwVEnp4zdfX%2BBu54foz6OXpQw%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Why should 18.x reject these foreign keys at restore time, given that a dump from an earlier 18.x release then restores without the foreign key?
2. Did you consider having 18.x reject only new definitions while still accepting them during restore or binary upgrade, with 19 and later rejecting them outright?
3. If 19.0 ships without the fix, will patch 0002 also go to REL_19_STABLE, or what else warns users with keys created on 19.0?
4. Could pg_dump warn about or flag such a foreign key, so that the user learns of the problem before the restore fails?
5. How often does the 18.x warning fire for one affected foreign key: once per RI plan build in each backend, or less often?

## Touch points

- **on-disk format / pg_upgrade:** Patch 0001 adds a pg_upgrade check in check.c for temporal foreign keys that reference non-unique indexes.
- **SQL-standard conflict or non-standard syntax:** The patch narrows which constraints a foreign key with PERIOD may reference; no new syntax.
- **pg_dump / restore round-trip:** A dump from 18.x that contains such a foreign key fails to restore that constraint on fixed versions.
- **backpatchability:** Patch 0001 is marked Backpatch-through: 18, and patch 0002 is for REL_18_STABLE only.
- **maintenance burden ("who maintains this in 10 years"):** The 18 branch carries warning code in ri_triggers.c and tablecmds.c, and pg_upgrade carries a new version-keyed check.
- **performance on the hot path:** Patch 0002 adds a check when the server builds an RI plan in ri_triggers.c, on 18 only.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the temporal key and RI trigger code and back-patching policy. Peter Eisentraut is listed as reviewer in the CommitFest entry but has not posted in the visible thread.
- Priority: proceed; audience niche; review cost low. Few users likely have such foreign keys, but those who do have referential integrity that is not reliably enforced. The two patches are small and touch tablecmds.c, ri_triggers.c and pg_upgrade's check.c; the open points are back-branch policy, which a committer can settle in review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
