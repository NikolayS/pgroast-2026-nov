# Design pre-review: pg_trgm: don't lose rows in index scans with a zero similarity threshold

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7382](https://commitfest.postgresql.org/patch/7382/); [thread](https://www.postgresql.org/message-id/flat/19701-c861a62e79bf49ce%40postgresql.org), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The proposal fixes a wrong-results bug in existing contrib opclasses, and no surviving objection questions the goal. The one surviving objection, standards-compat-1, is fixable: pgtrgm.sgml says "greater than" while the patch makes the indexes follow >=, and nobody has confirmed which meaning is intended. A committer who knows pg_trgm has to settle that and the back-patch decision.

Not checked: The code was not read, and the commit message and touched files of v3-0001 are not shown. Whether the consistent-function changes are correct for all strategies and for the GIN pending list could not be judged.

## Prior attempts

None found before 2026-09-19.

## Objections a reviewer is likely to raise

### 1. Patch fixes indexes to >= while pgtrgm.sgml still documents "greater than"

A user reads pgtrgm.sgml, which says % is true when similarity is "greater than" the threshold. The user sets the threshold to 0 to mean "any shared trigram". After the patch, every row comes back, including rows with similarity 0, and the documentation still describes the old meaning.

Message #1 states that the documented contract (>) and the implemented one (>=) differ, and that at threshold 0 this choice decides the whole outcome. The patch touches only trgm_gin.c, trgm_gist.c and tests, so it settles the semantics in code without changing the user-facing contract. The reporter's argument for >= is reasonable. But no reviewer or committer in the thread has confirmed it, and a committer would likely want the documentation changed in the same commit.

**Question:** Will the patch change pgtrgm.sgml to "greater than or equal to" for %, <% and <<%, and why is >= preferred over making 0 behave as documented?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/19701-c861a62e79bf49ce%40postgresql.org), [#2](https://www.postgresql.org/message-id/179019160285.4009503.14265883830439530659%40gmail.com), [#5](https://www.postgresql.org/message-id/CALfch1_qFgqr24XQY4kB93Gr62Jx%2BGUqLA0QYTpWsfrEZZ0wWg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Will the patch change pgtrgm.sgml to say "greater than or equal to" for %, <% and <<%, in the same commit?
2. Why is fixing the indexes to >= preferred over changing the operators to > so that threshold 0 behaves as documented?
3. Is the fix meant for back-patching to REL_14_STABLE and later, given that GIN results at threshold 0 change in a minor release?
4. What does v3-0001 touch, and should the calc_word_similarity() over-read fix be committed and back-patched separately from the index fix?
5. Should the documentation tell users that a GIN scan at threshold 0 reads the whole index?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The patch fixes the meaning of %, <% and <<% as >=, while pgtrgm.sgml documents "greater than".
- **planner behaviour change:** At threshold 0 a GIN scan becomes a full index scan, and the proposal does not discuss its cost.
- **backpatchability:** Message #2 says the patch applies to REL_14_STABLE through REL_19_STABLE; it changes query results at threshold 0 in minor releases.

## Who would commit it, and when

- Owner: silent. Someone who knows the pg_trgm opclasses and the GIN and GiST consistent-function contracts, including GIN search modes.
- Priority: needs champion; audience niche; review cost low. Few users set a threshold of 0, but the planner picks plans that silently lose rows for those who do. The patch is small and has one review, but no committer has replied, and the semantics and back-patch decisions need one.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
