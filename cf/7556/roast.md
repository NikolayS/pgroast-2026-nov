# Design pre-review: Uncancellable infinite loop in xpath() with entity references

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7556](https://commitfest.postgresql.org/patch/7556/); [thread](https://www.postgresql.org/message-id/flat/d3270e9f-4f10-4d04-8209-87e513801b05%40uni-muenster.de), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). No surviving objection questions the goal or the choice of XML_PARSE_NOENT, which xml_parse() and contrib/xml2 already use. Three fixable level 2 points remain: the branch plan for a result-changing bugfix (standards-compat-2), stale expression indexes and stored values after upgrade (standards-compat-1), and whether entity reference nodes can still reach XPath (maintenance-fit-1). Only message #1 is visible, so confidence is low.

Not checked: Without the code, nobody can confirm that XML_PARSE_HUGE stays off and that the external entity loader in xml.c covers these two parse calls. The claim that no entity reference nodes stay in the tree also needs a check against the code.

## Prior attempts

- [Added support of XML_PARSE_HUGE flag for XML documents](https://www.postgresql.org/message-id/flat/18274-98d16bc03520665f@postgresql.org) (related; Rejected, 2024-01-31). "The security team has discussed 2197d06224a1 after a report from coverity regarding the effects that issues like [1] would create in the backend, and concluded that this patch should be reverted because this could cause the backend to waste plently of CPU and/or memory even if the application applied checks on the size of the data given in input, and libxml2 does not offer guarantees that input limits are respected under XML_PARSE_HUGE." ([message](https://www.postgresql.org/message-id/ZbHlgrPLtBZyr_QW%40paquier.xyz)) Different now: P3 added XML_PARSE_HUGE, which lifts libxml2's size limits. This patch adds XML_PARSE_NOENT and keeps the default limits, and the row records that the earlier switch to XML_PARSE_NOENT stayed in.
- [Revert "Add support for parsing of large XML data (>= 10MB)"](https://www.postgresql.org/message-id/flat/ZbHlgrPLtBZyr_QW@paquier.xyz) (related; Reverted, 2024-01-26). This reverts commit 2197d06224a1, following a discussion over a Coverity report where issues like the "Billion laugh attack" could cause the backend to waste CPU and memory even if a client applied checks on the size of the data given in input, and libxml2 does not offer guarantees that input limits are respected under XML_PARSE_HUGE. Different now: P15 reverted XML_PARSE_HUGE because libxml2 gives no input-limit guarantees under that flag. This patch does not touch XML_PARSE_HUGE, so libxml2's default entity limits still apply.

## Objections a reviewer is likely to raise

### 1. Bugfix tag conflicts with a fix that changes query results

If the fix is back-patched, a minor release silently changes XMLTABLE and xpath() output for existing applications. If it is not back-patched, any user on a supported branch who sends a crafted document still gets a backend that ignores cancel and needs kill -9 and a restart.

The CommitFest entry is tagged Bugfix and targets 19, but message #1 does not say what happens on back branches. The fix for the hang also redefines which tree XPath sees, so two different changes are bundled together. A committer would likely want them separated: a narrow fix for the uncancellable loop that is safe to back-patch, and the semantic change done in master only, with documentation.

**Question:** What fix do back branches get for the uncancellable loop that leaves current xpath() and XMLTABLE results unchanged?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/d3270e9f-4f10-4d04-8209-87e513801b05%40uni-muenster.de); `standards-compat-2`</sub>

### 2. Changed xpath() results go stale in expression indexes and stored columns

A site has an expression index or a stored generated column on xpath() or xmlexists() over documents that declare internal entities. pg_upgrade carries the old index entries and stored values over unchanged. Index scans then disagree with recomputed results, for example missing rows whose text now reads "uy", until someone finds the cause and runs REINDEX.

Message #1 states that results change for documents with entity references. xpath() and xmlexists() are, as far as I know, marked immutable, so users may have built indexes and generated columns on them. The commit message lists no documentation change and no release-note or upgrade guidance. The claim that documents without entity references are not affected is true, but it does not help users who do have them.

**Question:** Will the patch document the change as an incompatibility and tell users with xpath() or xmlexists() indexes or generated columns to reindex or recompute after upgrade?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#1](https://www.postgresql.org/message-id/d3270e9f-4f10-4d04-8209-87e513801b05%40uni-muenster.de); `standards-compat-1`</sub>

### 3. Fix avoids one libxml2 loop but leaves backends uncancellable

A user runs a different XPath expression that hits another libxml2 traversal bug, or the same bug through a tree shape that NOENT does not remove. The backend again ignores cancel, statement_timeout and pg_terminate_backend(). The DBA finds out only when kill -9 forces a cluster restart.

Message #1 names the root cause as a probable bug in xmlXPathNextFollowing and says the investigation into interrupt handling inside libxml2 is still open. The patch removes the entity reference nodes that trigger this one case. It does not bound how long the backend can run inside libxml2 without checking for interrupts. A committer would likely ask whether the project is signing up to patch around each libxml2 traversal bug in turn, with nothing that fails loudly when a new one appears.

**Question:** Can xmlXPathNextFollowing loop on any tree without entity reference nodes, and is there a plan to keep backends cancellable during long libxml2 calls?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/d3270e9f-4f10-4d04-8209-87e513801b05%40uni-muenster.de); `maintenance-fit-1`</sub>

## Questions for the author

1. Which branches is the patch meant for, and what do back branches get for the uncancellable loop if the result change stays in master only?
2. Will the patch document the changed results as an incompatibility and tell users to reindex expression indexes and recompute stored columns built on xpath() or xmlexists()?
3. Can entity reference nodes still reach XPath evaluation after the patch, for example for references that libxml2 cannot resolve?
4. Which libxml2 versions show the loop in xmlXPathNextFollowing?
5. Is there a plan to keep backends cancellable during long libxml2 calls, or is that out of scope for this patch?

## Touch points

- **on-disk format / pg_upgrade:** Changed xpath() and xmlexists() results can differ from values already held in expression indexes and stored generated columns.
- **SQL-standard conflict or non-standard syntax:** xmlexists() and XMLTABLE are SQL/XML features, and their results change for documents with entity references.
- **security / privileges:** Any role that can call xpath() can start the uncancellable loop, and the patch adds entity expansion to two more parse calls.
- **backpatchability:** The entry is tagged Bugfix, but the fix changes query results and message #1 names no branches.
- **maintenance burden ("who maintains this in 10 years"):** The fix works around a probable libxml2 bug in xmlXPathNextFollowing, an interface the project does not control.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection maintenance-fit-1: Fix avoids one libxml2 loop but leaves backends uncancellable

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the libxml2 integration in xml.c and the back-patch policy for fixes that change results. None has replied in the visible thread.
- Priority: proceed; audience niche; review cost low. Few documents use internal entities, but any role that can call xpath() can force a cluster restart (message #1). The patch is 4.2 KiB, touches xml.c and its regression test, and reuses a flag xml_parse() already passes.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
