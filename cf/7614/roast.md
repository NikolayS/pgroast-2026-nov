# Design pre-review: XMLConcat() and XMLRoot() silently drop xml values with an invalid declaration

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7614](https://commitfest.postgresql.org/patch/7614/); [thread](https://www.postgresql.org/message-id/flat/757faa5a-dcfb-4878-8b94-aca25af17cc9%40uni-muenster.de), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The proposal states a reproducible bug and a narrow fix, and no objection survived at levels 1a or 1b. The one surviving objection, standards-compat-1, is a fixable level 2 point: the CommitFest entry targets stable branches, but message #1 names no branches and gives no repair step for values already stored. A committer who knows back-patch policy has to decide whether 0001's new errors fit a minor release.

Not checked: The code was not read. Whether the round-trip check in 0002 catches every VERSION that breaks the declaration, and whether 0001 covers all callers of parse_xml_decl(), cannot be judged from the prose.

## Prior attempts

None found before 2026-11-08.

## Objections a reviewer is likely to raise

### 1. New errors in a stable-branch bugfix need a stated back-patch scope

A site has one row with a bad declaration and runs a nightly report using xmlagg(x) over the table. After a minor upgrade the report fails with an error, where it used to return output with that row's content missing. The patch gives no repair step for that row.

The CommitFest entry targets "stable", but message #1 names no branches and does not discuss the new errors in a minor release. Message #1 argues that an error beats a wrong result, which is reasonable on master. In back branches, a committer would likely ask whether 0001 alone is enough, or whether 0002 alone could be back-patched with 0001 on master only. The patch also gives a query to find bad rows but no way to fix them.

**Question:** Which branches should get 0001 and 0002, and what documented step lets an affected user repair a bad row before or after the minor upgrade?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/757faa5a-dcfb-4878-8b94-aca25af17cc9%40uni-muenster.de); `standards-compat-1`</sub>

## Questions for the author

1. Which stable branches should get 0001, and which should get 0002?
2. What step lets a user repair a stored value with an unparsable declaration, before or after the minor upgrade?
3. Could 0002 alone go to the back branches, with 0001 on master only?
4. Do other callers of parse_xml_decl() also ignore its result?
5. Should xmlroot() also reject a VERSION that round-trips but is not a valid XML version, such as '2.0abc'?

## Touch points

- **on-disk format / pg_upgrade:** The format does not change, but stored xml values with an unparsable declaration stay on disk and now raise errors.
- **SQL-standard conflict or non-standard syntax:** xmlroot() is an SQL/XML function, and 0002 narrows the VERSION values it accepts.
- **pg_dump / restore round-trip:** 0002 stops new values that xml_in() rejects on restore; values already stored still cannot be restored.
- **backpatchability:** The CommitFest entry targets stable, and 0001 makes xmlconcat(), xmlagg() and xmlroot() raise errors where they returned a result.
- **maintenance burden ("who maintains this in 10 years"):** Small: the change stays in xml.c, but 0002 adds a round-trip rule for VERSION that the project then keeps.

## Who would commit it, and when

- Owner: silent. Someone who knows the xml type code in xml.c and the back-patch policy for behaviour changes in minor releases. Nobody has replied in the visible thread.
- Priority: proceed; audience niche; review cost low. Two small patches in xml.c with regression tests fix a wrong result and a dump that cannot be restored, for users who stored an xmlroot() value with an odd VERSION. The open point is the back-patch scope, which the thread can settle quickly.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
