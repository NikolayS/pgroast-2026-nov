# Design pre-review: XMLTable treats a column of a domain over xml as a non-xml column

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7609](https://commitfest.postgresql.org/patch/7609/) (Needs review); authors: Jim Jones (jimus)
- Thread: [XMLTable treats a column of a domain over xml as a non-xml column](https://www.postgresql.org/message-id/flat/394579a3-4913-4d10-aacc-e84fe8e10584%40uni-muenster.de), started 2026-11-07
- Messages read: 1 of 1
- Patch prose read: v1-0001-Treat-domains-over-xml-like-xml-in-XMLTABLE-colum.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection to the fix itself survived: looking through domains follows existing practice in XMLTABLE and in map_sql_value_to_xml_value(). The one surviving objection, standards-compat-1, is about the stable target. The patch changes query results and the value a domain CHECK constraint sees, and message #1 does not weigh that for minor releases. A committer decides whether to back-patch or fix master only.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. It would show whether domain constraints still run on the returned value, where the getBaseType() call sits, and whether other xml type tests need the same change.

## The proposal, restated

**Problem.** XMLTABLE handles a column declared as a domain over xml like a non-xml column, so a node-set result becomes its string value. In the example, an xml column returns `<a>x</a>` and a domain-over-xml column returns `x`. This affects users who declare XMLTABLE columns with such domains.

**Proposed design.** XmlTableGetValue() now resolves the column type to its base type with getBaseType() before it tests for XMLOID. A domain over xml then gets the same XML treatment as xml. The patch adds a regression test.

**User-visible surface.**
- XMLTABLE columns of a domain over xml now return the XML node, not its string value.

**What the author claims.**
- Both columns in the example should return the same value, since the domain is over xml.
- XMLTABLE already looks through domains for boolean results, which it maps by type category.
- Other code, such as map_sql_value_to_xml_value(), already treats domains like their base types.
- The fix belongs in stable branches, as the CommitFest entry targets stable and tags it as a bugfix.

**Thread state.** Only the first message is visible. No one has replied yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-07; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patching changes results of existing XMLTABLE queries in minor releases

fixable · level 2, rubric item 6 · backpatchability · `standards-compat-1`

A user has a view or report using XMLTABLE with a domain-over-xml column and gets the string value `x`. After a minor upgrade the same query returns `<a>x</a>`. Downstream code that compares, stores or parses that column breaks, and the user finds out only from wrong output in production.

The CommitFest entry targets stable and is tagged Bugfix, but the patch changes visible query results on stable branches. Message #1 argues the behaviour is inconsistent, yet the current output is well defined and may have users who rely on it. A committer would likely accept the fix for master only and ask for a release-note entry. Domain CHECK constraints now see a different value, so rows that passed before may fail after the upgrade, and the reverse can happen too.

Thread: [#1 (Jim Jones, 2026-11-07)](https://www.postgresql.org/message-id/394579a3-4913-4d10-aacc-e84fe8e10584%40uni-muenster.de)

**Question:** Why should a change in returned values, and in which values domain constraints see, go into minor releases and not only into master with a release note?

Counter-check: held. The CommitFest entry targets stable, and message #1 does not weigh the output change against users of the current result. The domain's input check would receive the node text instead of the string value, so the CHECK constraint point is plausible too.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | The patch changes what an SQL/XML XMLTABLE column of a domain over xml returns. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, and the patch changes results of existing XMLTABLE queries. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | XmlTableGetValue() gains a getBaseType() call when it converts a column value. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why should a change in XMLTABLE results for domain-over-xml columns go into minor releases, and not only into master with a release note?
2. Are the domain's CHECK constraints still checked on the returned value, and can a row that passed before the patch now fail?
3. Do other places in xml.c, such as the column case without PATH, have the same typid == XMLOID test?
4. Does XmlTableGetValue() call getBaseType() once per column or once per value?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the XMLTABLE code in xml.c and the back-patch policy for behaviour changes. No one has replied to message #1 yet.
- **Liability:** The committer signs for a result change in XMLTABLE on every branch that gets the patch. On stable branches this includes reports from users whose queries or domain constraints relied on the string value.
- **Author follow-up:** No sign either way; the thread has only message #1, posted four days before this review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 3.2 KB, touches xml.c and the xml regression test only, and affects only users who declare XMLTABLE columns as domains over xml. The open point is the back-patch decision.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Back-patching changes results of existing XMLTABLE queries on stable branches. Refuted (not specific): Duplicate of standards-compat-1, which makes the same back-patch point and adds the domain CHECK constraint case.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only retrieved row, [P3], is about domains and polymorphic functions, a different mechanism, and it has no recorded reason that could apply to XMLTABLE column handling. Message #1 does not mention an earlier discussion of this idea. Whether a back-patch is safe, given that existing queries would return different results, is a question for another angle.
- standards-compat: 1 objection(s). The master-only change fits existing practice for looking through domains. The objection is about back-patching only.
- replication-ondisk: 0 objection(s). The patch changes only how XmlTableGetValue() converts a result at query time, so it adds no on-disk format, WAL record, catalog entry, protocol message or extension API. Already stored data is unaffected, and the behaviour change on stable branches is a back-patch question for other angles.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how XMLTABLE builds a column value from a document the caller already passes, adds no new access, and widens nothing a user can see. Whether domain CHECK constraints still run on the returned xml value is a data-integrity question for other angles, and the code would show the answer.
- maintenance-fit: 1 objection(s). From the maintenance angle the fix is small, local to xml.c, and follows existing practice of looking through domains, so it adds no lasting cost. The only concern is the back-patch target.
- already-exists: 0 objection(s). Users can declare the column as xml and cast it to the domain in the select list, so the result is already reachable. This is a workaround for inconsistent behaviour rather than a competing feature, and no extension covers it, so a committer would not likely reject a small bugfix on these grounds.
- demand: 0 objection(s). This is a small consistency fix, so a demand test is the wrong bar: the cost falls only on users of domains over xml, and the change adds no syntax or GUC. No field report is shown. The workaround of declaring the column as xml exists, and whether a back-patch is safe belongs to another angle.

### Precedents retrieved

- P1: [pg_get_domain_ddl: DDL reconstruction function for CREATE DOMAIN statement](https://commitfest.postgresql.org/patch/6983/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P2: [Allow values to be cast to domain types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P3: [Make domains work better with polymorphic functions](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - used in the review
- P4: [Allow functions defined as casts to domains to be called during casting](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Domains) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P5: [loop over unbound cursor](https://commitfest.postgresql.org/patch/2558/) (cf, Withdrawn, 2020-08-10) - judged not relevant; reason summarised from its thread
- P6: [Fix standby may crash when switching-over in certain special cases](https://commitfest.postgresql.org/patch/5199/) (cf, Withdrawn, 2026-09-07) - judged not relevant; reason summarised from its thread
- P7: [iteration over record in plpgsql](https://commitfest.postgresql.org/patch/124/) (cf, Returned with feedback, 2015-07-07) - judged not relevant; reason summarised from its thread
- P8: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - judged not relevant; reason summarised from its thread
- P9: [refactor AlterDomainAddConstraint (alter domain add constraint)](https://commitfest.postgresql.org/patch/5455/) (cf, Returned with feedback, 2026-04-02) - judged not relevant; reason summarised from its thread
- P10: [Revert: Avoid looping over all type cache entries in TypeCacheRelCallback()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8daa62a10c911c851f7e9ec5ef7b90cfd4b73212) (revert, Reverted, 2024-08-25) - judged not relevant
- P11: [Restructure XML and /contrib/xml2 functionality](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P12: [Document AccessExclusive lock behaviour on standbys](https://commitfest.postgresql.org/patch/5920/) (cf, Withdrawn, 2026-02-09) - judged not relevant; reason summarised from its thread
- P13: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P14: [Fix to_char() behaviour with 'WW-D' pattern](https://commitfest.postgresql.org/patch/2388/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P15: [Revert "Add USER SET parameter values for pg_db_role_setting"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b9a7a822723aebb16cbe7e5fb874e5124745b07e) (revert, Reverted, 2023-05-17) - judged not relevant
- P16: [Enable fine-grained control over what gets logged on connection attempt (reduces log size)](https://commitfest.postgresql.org/patch/4005/) (cf, Returned with feedback, 2023-08-03) - judged not relevant; reason summarised from its thread
- P17: [Add GUC to tune glibc's malloc behaviour](https://commitfest.postgresql.org/patch/4387/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P18: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - judged not relevant
- P19: [function pg_setting_value_split() to parse shared_preload_libraries etc.](https://commitfest.postgresql.org/patch/2782/) (cf, Withdrawn, 2020-11-29) - judged not relevant
- P20: [Fix to_number() handling for values not matching the format string](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Character_Formatting) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P21: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant
- P22: [Allow XML arrays to be cast to other data types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P23: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P24: [Use variadic macros for syscache lookup functions](https://commitfest.postgresql.org/patch/4261/) (cf, Withdrawn, 2023-11-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 103 s; cost reported by the CLI: $0.85
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 52 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:06:45.395Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
