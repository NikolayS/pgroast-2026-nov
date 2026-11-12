# Design pre-review: XMLTable treats a column of a domain over xml as a non-xml column

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7609](https://commitfest.postgresql.org/patch/7609/); [thread](https://www.postgresql.org/message-id/flat/394579a3-4913-4d10-aacc-e84fe8e10584%40uni-muenster.de), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection to the fix itself survived: looking through domains follows existing practice in XMLTABLE and in map_sql_value_to_xml_value(). The one surviving objection, standards-compat-1, is about the stable target. The patch changes query results and the value a domain CHECK constraint sees, and message #1 does not weigh that for minor releases. A committer decides whether to back-patch or fix master only.

Not checked: The code was not read. It would show whether domain constraints still run on the returned value, where the getBaseType() call sits, and whether other xml type tests need the same change.

## Prior attempts

None found before 2026-11-07.

## Objections a reviewer is likely to raise

### 1. Back-patching changes results of existing XMLTABLE queries in minor releases

A user has a view or report using XMLTABLE with a domain-over-xml column and gets the string value `x`. After a minor upgrade the same query returns `<a>x</a>`. Downstream code that compares, stores or parses that column breaks, and the user finds out only from wrong output in production.

The CommitFest entry targets stable and is tagged Bugfix, but the patch changes visible query results on stable branches. Message #1 argues the behaviour is inconsistent, yet the current output is well defined and may have users who rely on it. A committer would likely accept the fix for master only and ask for a release-note entry. Domain CHECK constraints now see a different value, so rows that passed before may fail after the upgrade, and the reverse can happen too.

**Question:** Why should a change in returned values, and in which values domain constraints see, go into minor releases and not only into master with a release note?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/394579a3-4913-4d10-aacc-e84fe8e10584%40uni-muenster.de); `standards-compat-1`</sub>

## Questions for the author

1. Why should a change in XMLTABLE results for domain-over-xml columns go into minor releases, and not only into master with a release note?
2. Are the domain's CHECK constraints still checked on the returned value, and can a row that passed before the patch now fail?
3. Do other places in xml.c, such as the column case without PATH, have the same typid == XMLOID test?
4. Does XmlTableGetValue() call getBaseType() once per column or once per value?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The patch changes what an SQL/XML XMLTABLE column of a domain over xml returns.
- **backpatchability:** The CommitFest entry targets stable, and the patch changes results of existing XMLTABLE queries.
- **performance on the hot path:** XmlTableGetValue() gains a getBaseType() call when it converts a column value.

## Who would commit it, and when

- Owner: silent. Someone who knows the XMLTABLE code in xml.c and the back-patch policy for behaviour changes. No one has replied to message #1 yet.
- Priority: proceed; audience niche; review cost low. The patch is 3.2 KB, touches xml.c and the xml regression test only, and affects only users who declare XMLTABLE columns as domains over xml. The open point is the back-patch decision.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
