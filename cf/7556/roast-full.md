# Design pre-review: Uncancellable infinite loop in xpath() with entity references

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7556](https://commitfest.postgresql.org/patch/7556/) (Needs review); authors: Jim Jones (jimus)
- Thread: [Uncancellable infinite loop in xpath() with entity references](https://www.postgresql.org/message-id/flat/d3270e9f-4f10-4d04-8209-87e513801b05%40uni-muenster.de), started 2026-10-29
- Messages read: 1 of 1
- Patch prose read: v1-0001-Expand-entity-references-in-xpath-xmlexists-and-X.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

No surviving objection questions the goal or the choice of XML_PARSE_NOENT, which xml_parse() and contrib/xml2 already use. Three fixable level 2 points remain: the branch plan for a result-changing bugfix (standards-compat-2), stale expression indexes and stored values after upgrade (standards-compat-1), and whether entity reference nodes can still reach XPath (maintenance-fit-1). Only message #1 is visible, so confidence is low.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that XML_PARSE_HUGE stays off and that the external entity loader in xml.c covers these two parse calls. The claim that no entity reference nodes stay in the tree also needs a check against the code.

## The proposal, restated

**Problem.** A query such as xpath('/r/text()/following::comment()', ...) on a document with an entity reference loops forever inside libxml2. The backend ignores cancel, statement_timeout and pg_terminate_backend(), so only kill -9 and a restart stop it. Separately, xpath() sees a different tree than the rest of the xml code, so count(//b) returns 0 where the expanded document has 2.

**Proposed design.** The patch adds XML_PARSE_NOENT to the parse calls in xpath_internal() and XmlTableSetDocument(). xml_parse() and contrib/xml2 already use this flag. Internal entities are then expanded before XPath evaluation, so no entity reference nodes stay in the tree.

**User-visible surface.**
- xpath() returns different results for documents with entity references, e.g. /r/text() on <r>u&e;</r> gives {uy} instead of {u}.
- xmlexists() evaluates against the entity-expanded document.
- XMLTABLE evaluates against the entity-expanded document.

**What the author claims.**
- The change fixes the uncancellable infinite loop.
- xpath(), xmlexists() and XMLTABLE become consistent with xml_parse() and contrib/xml2.
- The changed results are arguably an improvement.
- Documents without entity references are not affected.
- The root cause is most likely a libxml2 bug in xmlXPathNextFollowing, which the author plans to report.

**Thread state.** Only the first message is visible. The entry is tagged as a bugfix, targets version 19, and needs review.

## 2. Prior attempts

### Added support of XML_PARSE_HUGE flag for XML documents (related)

- **Link:** [https://commitfest.postgresql.org/patch/4772/](https://commitfest.postgresql.org/patch/4772/)
- **Thread:** [https://www.postgresql.org/message-id/flat/18274-98d16bc03520665f@postgresql.org](https://www.postgresql.org/message-id/flat/18274-98d16bc03520665f@postgresql.org)
- **Date:** decided 2024-01-31; started 2024-01-06
- **Outcome:** Rejected (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The XML_PARSE_HUGE change was committed to HEAD as 2197d06224a1 and then reverted in f2743a7d70e7. The security team found that it lets the backend use large amounts of CPU and memory in cases like the billion laughs attack, even when the application checks input size, because libxml2 does not guarantee input limits under XML_PARSE_HUGE. The author agreed with the revert. The separate replacement of xmlSubstituteEntitiesDefault() with XML_PARSE_NOENT (65c5864d7fac) stayed in.
- **Reason (quote):** "The security team has discussed 2197d06224a1 after a report from coverity regarding the effects that issues like [1] would create in the backend, and concluded that this patch should be reverted because this could cause the backend to waste plently of CPU and/or memory even if the application applied checks on the size of the data given in input, and libxml2 does not offer guarantees that input limits are respected under XML_PARSE_HUGE." ([message](https://www.postgresql.org/message-id/ZbHlgrPLtBZyr_QW%40paquier.xyz))
- **What is different this time:** P3 added XML_PARSE_HUGE, which lifts libxml2's size limits. This patch adds XML_PARSE_NOENT and keeps the default limits, and the row records that the earlier switch to XML_PARSE_NOENT stayed in.

### Revert "Add support for parsing of large XML data (>= 10MB)" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f2743a7d70e7b2891277632121bb51e739743a47](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f2743a7d70e7b2891277632121bb51e739743a47)
- **Thread:** [https://www.postgresql.org/message-id/flat/ZbHlgrPLtBZyr_QW@paquier.xyz](https://www.postgresql.org/message-id/flat/ZbHlgrPLtBZyr_QW@paquier.xyz), [https://www.postgresql.org/message-id/flat/18274-98d16bc03520665f@postgresql.org](https://www.postgresql.org/message-id/flat/18274-98d16bc03520665f@postgresql.org)
- **Date:** decided 2024-01-26; started 2024-01-17
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 2197d06224a1, following a discussion over a Coverity report where issues like the "Billion laugh attack" could cause the backend to waste CPU and memory even if a client applied checks on the size of the data given in input, and libxml2 does not offer guarantees that input limits are respected under XML_PARSE_HUGE.
- **Reason (quote):** quote unavailable
- **What is different this time:** P15 reverted XML_PARSE_HUGE because libxml2 gives no input-limit guarantees under that flag. This patch does not touch XML_PARSE_HUGE, so libxml2's default entity limits still apply.


## 3. Objections a reviewer is likely to raise

### 1. Bugfix tag conflicts with a fix that changes query results

fixable · level 2, rubric item 6 · backpatchability · `standards-compat-2`

If the fix is back-patched, a minor release silently changes XMLTABLE and xpath() output for existing applications. If it is not back-patched, any user on a supported branch who sends a crafted document still gets a backend that ignores cancel and needs kill -9 and a restart.

The CommitFest entry is tagged Bugfix and targets 19, but message #1 does not say what happens on back branches. The fix for the hang also redefines which tree XPath sees, so two different changes are bundled together. A committer would likely want them separated: a narrow fix for the uncancellable loop that is safe to back-patch, and the semantic change done in master only, with documentation.

Thread: [#1 (Jim Jones, 2026-10-29)](https://www.postgresql.org/message-id/d3270e9f-4f10-4d04-8209-87e513801b05%40uni-muenster.de)

**Question:** What fix do back branches get for the uncancellable loop that leaves current xpath() and XMLTABLE results unchanged?

Counter-check: held. The entry is tagged Bugfix, and message #1 says results change but does not name the branches. The material does not show that a narrower fix exists, so the request to split the patch is a question for the author, not a requirement.

### 2. Changed xpath() results go stale in expression indexes and stored columns

fixable · level 2, rubric item 6 · on-disk format / pg_upgrade, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A site has an expression index or a stored generated column on xpath() or xmlexists() over documents that declare internal entities. pg_upgrade carries the old index entries and stored values over unchanged. Index scans then disagree with recomputed results, for example missing rows whose text now reads "uy", until someone finds the cause and runs REINDEX.

Message #1 states that results change for documents with entity references. xpath() and xmlexists() are, as far as I know, marked immutable, so users may have built indexes and generated columns on them. The commit message lists no documentation change and no release-note or upgrade guidance. The claim that documents without entity references are not affected is true, but it does not help users who do have them.

Thread: [#1 (Jim Jones, 2026-10-29)](https://www.postgresql.org/message-id/d3270e9f-4f10-4d04-8209-87e513801b05%40uni-muenster.de)

**Question:** Will the patch document the change as an incompatibility and tell users with xpath() or xmlexists() indexes or generated columns to reindex or recompute after upgrade?

Counter-check: held. xpath() and xmlexists() are immutable, and stored xml values keep their entity references, so an expression index over such documents can hold old results after upgrade. The case is rare, but the patch has no documentation or upgrade note for it.

### 3. Fix avoids one libxml2 loop but leaves backends uncancellable

fixable · level 2, rubric item 15 · maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-1`

A user runs a different XPath expression that hits another libxml2 traversal bug, or the same bug through a tree shape that NOENT does not remove. The backend again ignores cancel, statement_timeout and pg_terminate_backend(). The DBA finds out only when kill -9 forces a cluster restart.

Message #1 names the root cause as a probable bug in xmlXPathNextFollowing and says the investigation into interrupt handling inside libxml2 is still open. The patch removes the entity reference nodes that trigger this one case. It does not bound how long the backend can run inside libxml2 without checking for interrupts. A committer would likely ask whether the project is signing up to patch around each libxml2 traversal bug in turn, with nothing that fails loudly when a new one appears.

Thread: [#1 (Jim Jones, 2026-10-29)](https://www.postgresql.org/message-id/d3270e9f-4f10-4d04-8209-87e513801b05%40uni-muenster.de)

**Question:** Can xmlXPathNextFollowing loop on any tree without entity reference nodes, and is there a plan to keep backends cancellable during long libxml2 calls?

Counter-check: held. The fix depends on no entity reference nodes staying in the tree, and the material does not show this for references libxml2 cannot resolve. The general worry about other libxml2 bugs has no concrete scenario and should not carry the objection.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Changed xpath() and xmlexists() results can differ from values already held in expression indexes and stored generated columns. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | xmlexists() and XMLTABLE are SQL/XML features, and their results change for documents with entity references. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Any role that can call xpath() can start the uncancellable loop, and the patch adds entity expansion to two more parse calls. |
| backpatchability | **yes** | The entry is tagged Bugfix, but the fix changes query results and message #1 names no branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The fix works around a probable libxml2 bug in xmlXPathNextFollowing, an interface the project does not control. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection maintenance-fit-1: Fix avoids one libxml2 loop but leaves backends uncancellable |

## 5. Questions the author must answer before anyone reviews code

1. Which branches is the patch meant for, and what do back branches get for the uncancellable loop if the result change stays in master only?
2. Will the patch document the changed results as an incompatibility and tell users to reindex expression indexes and recompute stored columns built on xpath() or xmlexists()?
3. Can entity reference nodes still reach XPath evaluation after the patch, for example for references that libxml2 cannot resolve?
4. Which libxml2 versions show the loop in xmlXPathNextFollowing?
5. Is there a plan to keep backends cancellable during long libxml2 calls, or is that out of scope for this patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the libxml2 integration in xml.c and the back-patch policy for fixes that change results. None has replied in the visible thread.
- **Liability:** The owner signs for changed xpath(), xmlexists() and XMLTABLE results on documents with entity references, for the branch decision, and for any libxml2 loop this workaround does not cover.
- **Author follow-up:** The author says in message #1 that he will investigate the libxml2 bug and report it. There is no other sign either way.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Few documents use internal entities, but any role that can call xpath() can force a cluster restart (message #1). The patch is 4.2 KiB, touches xml.c and its regression test, and reuses a flag xml_parse() already passes.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Changed xpath() results leave existing expression indexes inconsistent. Refuted (not specific): Duplicate of standards-compat-1. The added point about mixed minor versions in physical replication depends on a backpatch that the proposal does not ask for.
- `security-1` Unprivileged backend hang stays in back branches after this fix. Refuted (not specific): Duplicate of standards-compat-2 for the back-branch question and of maintenance-fit-1 for the missing interrupt handling inside libxml2. The privilege point adds weight to those two but no separate failure.
- `maintenance-fit-2` Bugfix tag conflicts with changed query results in back branches. Refuted (not specific): Duplicate of standards-compat-2, which states the same back-branch conflict and says what a committer would ask for.

### Review angles

- prior-attempts: 0 objection(s). No precedent records a rejection of XML_PARSE_NOENT. The only related reason, entity-expansion DoS, was tied to XML_PARSE_HUGE, which this patch does not add. xml_parse() already expands entities with this flag, as message #1 shows, so the reason does not carry over. Back-branch policy for the changed results is a question for other angles.
- standards-compat: 2 objection(s). Two fixable objections. Aligning with xml_parse() and contrib/xml2 matches existing practice, so consistency counts in the patch's favour.
- replication-ondisk: 1 objection(s). The patch touches no WAL, catalog, wire protocol or ABI. Its only persistent effect is through stored or indexed results of immutable functions, which the objection covers.
- security: 1 objection(s). External entities and expansion size give no objection, because xml input already parses with XML_PARSE_NOENT, so expanding entities again in xpath() adds no new exposure. This assumes the patch keeps XML_PARSE_HUGE off, unlike [P3] and [P15], and that the external entity loader in xml.c still covers these parse calls; both need confirming in the code.
- maintenance-fit: 2 objection(s). The core change copies existing practice in xml_parse() and contrib/xml2, so a separate fit objection is weak. Input documents already pass entity expansion in xml_in, which limits any new expansion cost on these paths.
- already-exists: 0 objection(s). No existing feature does this job. Re-parsing through xmlserialize gives the expanded tree, but it does not stop any user from hanging an uncancellable backend with plain xpath(). The patch reuses the flag xml_parse() and contrib/xml2 already use, so it extends existing practice and adds no second mechanism.
- demand: 0 objection(s). This is a bugfix, so field demand is the wrong test. Any user who can run xpath() on a short crafted document can start a backend loop that ignores cancel, statement_timeout and pg_terminate_backend(), and only a restart stops it (message #1). The changed results affect only documents with entity references, so their users bear the cost. Whether the fix should be backpatched, and whether XML_PARSE_NOENT is safe here (see [P3], [P15]), belong to other angles.

### Precedents retrieved

- P1: [Add table expansion hook](https://commitfest.postgresql.org/patch/2614/) (cf, Returned with feedback, 2021-09-13) - judged not relevant; reason summarised from its thread
- P2: [Add "\gi file" to psql to allow COPY with variable expansion](https://commitfest.postgresql.org/patch/5674/) (cf, Returned with feedback, 2025-06-17) - judged not relevant; reason summarised from its thread
- P3: [Added support of XML_PARSE_HUGE flag for XML documents](https://commitfest.postgresql.org/patch/4772/) (cf, Rejected, 2024-01-31) - used in the review; reason summarised from its thread
- P4: [XML XPath default namespace support](https://commitfest.postgresql.org/patch/1085/) (cf, Rejected, 2019-02-04) - judged not relevant
- P5: [Resume vacuum and autovacuum from interruption and cancellation](https://commitfest.postgresql.org/patch/2211/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P6: [Improve handling of PIs and DTDs in xmlconcat() \[5\]](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Allow reliable XML operation non-UTF8 server encodings (xpath(), in particular, is known to not work)](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [xpath_table is pretty broken anyway \[2\]](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [pgbench: allow to cancel queries during benchmark](https://commitfest.postgresql.org/patch/4399/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P10: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant; reason summarised from its thread
- P11: [Infinite Autovacuum loop caused by failing virtual generated column expression](https://commitfest.postgresql.org/patch/7100/) (cf, Withdrawn, 2026-08-19) - judged not relevant; reason summarised from its thread
- P12: [Cancel problems of query to pg_stat_statements](https://commitfest.postgresql.org/patch/5825/) (cf, Rejected, 2026-03-25) - judged not relevant; reason summarised from its thread
- P13: [xpath_table needs to be implemented/implementable to get rid of contrib/xml2 \[1\]](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P14: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant; reason summarised from its thread
- P15: [Revert "Add support for parsing of large XML data (>= 10MB)"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f2743a7d70e7b2891277632121bb51e739743a47) (revert, Reverted, 2024-01-26) - used in the review
- P16: [Support for 8-byte TOAST values (aka the TOAST infinite loop problem)](https://commitfest.postgresql.org/patch/5830/) (cf, Returned with feedback, 2026-02-10) - judged not relevant
- P17: [rename column if exists](https://commitfest.postgresql.org/patch/3045/) (cf, Rejected, 2022-01-17) - judged not relevant
- P18: [OOM in libpq and infinite loop with getCopyStart()](https://commitfest.postgresql.org/patch/565/) (cf, Returned with feedback, 2017-01-23) - judged not relevant
- P19: [Do not exit on postmaster death event inside CRIT sections.](https://commitfest.postgresql.org/patch/5964/) (cf, Returned with feedback, 2025-08-28) - judged not relevant
- P20: [Cleanup gcc trick with varattrib_1b_e in VARATT_EXTERNAL_GET_POINTER()](https://commitfest.postgresql.org/patch/5807/) (cf, Withdrawn, 2025-07-14) - judged not relevant
- P21: [CREATE ROLE IF NOT EXISTS](https://commitfest.postgresql.org/patch/3370/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P22: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant
- P23: [Support external parameters in EXECUTE command](https://commitfest.postgresql.org/patch/2460/) (cf, Returned with feedback, 2020-04-05) - judged not relevant
- P24: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 110 s; cost reported by the CLI: $1.08
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 79 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:09:42.771Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
