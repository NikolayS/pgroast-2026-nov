# Design pre-review: XMLConcat() and XMLRoot() silently drop xml values with an invalid declaration

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7614](https://commitfest.postgresql.org/patch/7614/) (Needs review); authors: Jim Jones (jimus)
- Thread: [XMLConcat() and XMLRoot() silently drop xml values with an invalid declaration](https://www.postgresql.org/message-id/flat/757faa5a-dcfb-4878-8b94-aca25af17cc9%40uni-muenster.de), started 2026-11-08
- Messages read: 1 of 1
- Patch prose read: v1-0001-Raise-an-error-for-invalid-XML-declarations-in-xm.patch, v1-0002-Reject-VERSION-values-in-xmlroot-that-break-the-X.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The proposal states a reproducible bug and a narrow fix, and no objection survived at levels 1a or 1b. The one surviving objection, standards-compat-1, is a fixable level 2 point: the CommitFest entry targets stable branches, but message #1 names no branches and gives no repair step for values already stored. A committer who knows back-patch policy has to decide whether 0001's new errors fit a minor release.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Whether the round-trip check in 0002 catches every VERSION that breaks the declaration, and whether 0001 covers all callers of parse_xml_decl(), cannot be judged from the prose.

## The proposal, restated

**Problem.** xmlroot() copies VERSION into the XML declaration without checking it, so it can store values like '1.0"x' that xml_in() rejects. xmlconcat(), xmlagg() and xmlroot() then silently drop the content of such values, and a dump of them cannot be restored. This affects anyone who stored such a value.

**Proposed design.** 0001 makes xmlconcat() and xmlroot() check the result of parse_xml_decl() and raise the error that xml_out() already reports as a warning, so they no longer drop content. 0002 makes xmlroot() reject a VERSION that cannot be read back unchanged from the declaration it builds. It does no other validity check on the version.

**User-visible surface.**
- xmlconcat() and xmlagg() raise an error for a stored value with an unparsable XML declaration, where they returned a result without its content.
- xmlroot() raises the same error for such input values.
- xmlroot() rejects a VERSION containing a quote or a non-ASCII character, with error "invalid XML version".

**What the author claims.**
- An error is better than the wrong result that queries return today.
- Affected rows can be found with xml_is_well_formed_content(x::text).
- 0002 prevents new values of this kind from being created.

**Thread state.** Only the first message is visible. Nobody has replied yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. New errors in a stable-branch bugfix need a stated back-patch scope

fixable · level 2, rubric item 6 · backpatchability, pg_dump / restore round-trip · `standards-compat-1`

A site has one row with a bad declaration and runs a nightly report using xmlagg(x) over the table. After a minor upgrade the report fails with an error, where it used to return output with that row's content missing. The patch gives no repair step for that row.

The CommitFest entry targets "stable", but message #1 names no branches and does not discuss the new errors in a minor release. Message #1 argues that an error beats a wrong result, which is reasonable on master. In back branches, a committer would likely ask whether 0001 alone is enough, or whether 0002 alone could be back-patched with 0001 on master only. The patch also gives a query to find bad rows but no way to fix them.

Thread: [#1 (Jim Jones, 2026-11-08)](https://www.postgresql.org/message-id/757faa5a-dcfb-4878-8b94-aca25af17cc9%40uni-muenster.de)

**Question:** Which branches should get 0001 and 0002, and what documented step lets an affected user repair a bad row before or after the minor upgrade?

Counter-check: held. The CommitFest entry targets stable, and message #1 names no branches and gives no repair step for stored values. Message #1 argues an error beats a wrong result, but it does not weigh that for a minor release.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The format does not change, but stored xml values with an unparsable declaration stay on disk and now raise errors. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | xmlroot() is an SQL/XML function, and 0002 narrows the VERSION values it accepts. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | 0002 stops new values that xml_in() rejects on restore; values already stored still cannot be restored. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, and 0001 makes xmlconcat(), xmlagg() and xmlroot() raise errors where they returned a result. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Small: the change stays in xml.c, but 0002 adds a round-trip rule for VERSION that the project then keeps. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which stable branches should get 0001, and which should get 0002?
2. What step lets a user repair a stored value with an unparsable declaration, before or after the minor upgrade?
3. Could 0002 alone go to the back branches, with 0001 on master only?
4. Do other callers of parse_xml_decl() also ignore its result?
5. Should xmlroot() also reject a VERSION that round-trips but is not a valid XML version, such as '2.0abc'?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the xml type code in xml.c and the back-patch policy for behaviour changes in minor releases. Nobody has replied in the visible thread.
- **Liability:** The committer signs for new errors in minor releases on queries that returned a result before, and for the VERSION rule in xmlroot() from then on. The code is small and stays in xml.c.
- **Author follow-up:** No sign either way; the author wrote both patches and asks for comments in message #1.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Two small patches in xml.c with regression tests fix a wrong result and a dump that cannot be restored, for users who stored an xmlroot() value with an odd VERSION. The open point is the back-patch scope, which the thread can settle quickly.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Back-patching 0001 turns queries on already stored values into errors. Refuted (not specific): Duplicate of standards-compat-1. Its point that existing values still cannot be restored from a dump is correct, but it leads to the same request for branches and a repair step.
- `maintenance-fit-1` Back-patching 0001 turns wrong results into errors on minor upgrade. Refuted (not specific): Duplicate of standards-compat-1. It uses the same xmlagg() report scenario and asks the same question about branches and repair.
- `maintenance-fit-2` Round-trip check in 0002 is an ad hoc rule for VERSION. Refuted (no failure scenario): xmlroot() accepts '2.0abc' today, so 0002 only narrows the accepted set and a later grammar check would need the same compatibility change without it. The commit message states the limit, and a narrow check suits a fix aimed at stable branches.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only retrieved row, [P7], is a TODO item about PIs and DTDs in xmlconcat(). This proposal is about unparsable XML declarations, and [P7] records no reason that would carry over. Message #1 does not point to any earlier discussion of xmlroot() VERSION checks or of xmlconcat() dropping content.
- standards-compat: 1 objection(s). The user-facing contract otherwise fits: VERSION in a declaration cannot hold a quote, and the error text reuses the one xml_out() already gives. [P7] concerns PIs and DTDs in xmlconcat() and has no recorded reason, so it does not apply.
- replication-ondisk: 1 objection(s). The patch changes no on-disk format, WAL, catalog, protocol or ABI. The only persistent issue is what happens to xml values already stored in the bad form.
- security: 0 objection(s). No security objection. The patch adds no privilege, role or new exposure, and its errors report parse failures without value content. 0002 removes a way for any user with INSERT on an xml column to create values that break pg_dump restore, which reduces exposure.
- maintenance-fit: 2 objection(s). There is no hot-path cost and no new API. The change stays inside xml.c. P7 concerns PIs and DTDs in xmlconcat(), has no recorded reason, and does not apply.
- already-exists: 0 objection(s). No existing feature, extension or idiom gives a correct xmlconcat() or xmlagg() result for these values, and nothing in Postgres stops xmlroot() from storing a VERSION that breaks the declaration. This is a bug fix inside existing functions and adds no second mechanism. [P7] is about PIs and DTDs, and its recorded reason does not apply here.
- demand: 0 objection(s). This is a correctness fix: xmlconcat() and xmlagg() drop content, and xmlroot() creates values that a dump cannot restore. Bug fixes like this need no demand evidence, and there is no field report to weigh. Whether the new errors belong in stable branches is a back-patch question for another angle. [P7] proposed a feature change to xmlconcat(), not a bug fix, so it does not apply.

### Precedents retrieved

- P1: [Added support of XML_PARSE_HUGE flag for XML documents](https://commitfest.postgresql.org/patch/4772/) (cf, Rejected, 2024-01-31) - judged not relevant; reason summarised from its thread
- P2: [Declarative partitioning](https://commitfest.postgresql.org/patch/331/) (cf, Returned with feedback, 2015-12-02) - judged not relevant; reason summarised from its thread
- P3: [Declarative partitioning](https://commitfest.postgresql.org/patch/469/) (cf, Returned with feedback, 2016-03-21) - judged not relevant; reason summarised from its thread
- P4: [ECPG: Declare Statement](https://commitfest.postgresql.org/patch/936/) (cf, Returned with feedback, 2017-11-16) - judged not relevant; reason summarised from its thread
- P5: [Revert "Add DECLARE STATEMENT support to ECPG."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96b6c82c9dd4a6a91c7e54bf42d36da111959ec6) (revert, Reverted, 2019-09-20) - judged not relevant
- P6: [Allow XML arrays to be cast to other data types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Improve handling of PIs and DTDs in xmlconcat() \[5\]](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - used in the review
- P8: [Restructure XML and /contrib/xml2 functionality](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [Revert "Reject degenerate SPLIT PARTITION with DEFAULT partition"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0392fb900eb89f52988cccd33046443c39c70d1c) (revert, Reverted, 2026-05-20) - judged not relevant
- P10: [Dump-restore loosing 'attnotnull' bit for DEFERRABLE PRIMARY KEY column(s)](https://commitfest.postgresql.org/patch/4729/) (cf, Rejected, 2023-12-28) - judged not relevant; reason summarised from its thread
- P11: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P12: [Bug: wrong relname in RemoveSubscriptionRel() error detail](https://commitfest.postgresql.org/patch/6621/) (cf, Withdrawn, 2026-07-12) - judged not relevant; reason summarised from its thread
- P13: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P14: [add DETAIL for wrong SCRAM password](https://commitfest.postgresql.org/patch/3022/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P15: [Revert "Add support for parsing of large XML data (>= 10MB)"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f2743a7d70e7b2891277632121bb51e739743a47) (revert, Reverted, 2024-01-26) - judged not relevant
- P16: [Revert change of ecpglib major version](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74f2a8aa27cb7bd2dbab3ed58e6b5c239a6b4a31) (revert, Reverted, 2019-09-19) - judged not relevant
- P17: [doc: a restore executes code chosen by any dumped object's owner](https://commitfest.postgresql.org/patch/7185/) (cf, Withdrawn, 2026-10-01) - judged not relevant; reason summarised from its thread
- P18: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant
- P19: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [to_date_valid()](https://commitfest.postgresql.org/patch/662/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P21: [\[PG19\]pg_dump --statistics restores extended statistics by attribute number](https://commitfest.postgresql.org/patch/7446/) (cf, Withdrawn, 2026-10-19) - judged not relevant
- P22: [COPY: validate option presence rather than option values](https://commitfest.postgresql.org/patch/6744/) (cf, Withdrawn, 2026-06-29) - judged not relevant
- P23: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P24: [Implement TODOs for index operator strategy number validation](https://commitfest.postgresql.org/patch/7152/) (cf, Withdrawn, 2026-09-09) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 111 s; cost reported by the CLI: $1.03
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 64 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:07:36.110Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
