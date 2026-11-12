# Design pre-review: Validate JSON object keys before omitting null values (BUG #19736)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7605](https://commitfest.postgresql.org/patch/7605/); [thread](https://www.postgresql.org/message-id/flat/19736-3a44617ea5534c77%40postgresql.org), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the goal of consistent key validation is clear. standards-compat-1 and standards-compat-2 are fixable points on back-patch scope, a release note and the SQL standard's rule for omitted pairs. A committer who knows SQL/JSON has to decide those, and none has spoken in the visible thread.

Not checked: The code was not read, so the claim that omitted keys never reach output or cast functions is unchecked. Only the v1 commit message and 3 of 8 messages were available, so the v2 helper and documentation text could not be judged.

## Prior attempts

- [COPY: validate option presence rather than option values](https://www.postgresql.org/message-id/flat/C1D2509E-E5D1-46B0-932C-B57AA7B963A1@gmail.com) (related; Withdrawn, 2026-06-29). "My concern is that the proposed patch could make the command unnecessarily less flexible." ([message](https://www.postgresql.org/message-id/CAD21AoCsbjvgk5tOarepYF28dW7P6NG8SLVDQ72A7AJ3yEjriw%40mail.gmail.com)) Different now: P11 rejected COPY options by their presence instead of their value. This patch rejects JSON keys whose pair is dropped anyway. Both add checks on input that does not affect the result.
- [Revert cascading of JSON_TABLE's ON ERROR](https://www.postgresql.org/message-id/flat/CAA-aLv7aZGSExnbjJRw8eKkoXbu34TdoKLLA2gPye3aHjO5OSA@mail.gmail.com) (related; Reverted, 2026-07-14). 86ab7f4c721d commit made the table-level ON ERROR clause serve as the default ON ERROR for columns lacking their own, so that a top-level ERROR ON ERROR turned per-column evaluation errors into hard errors. The SQL standard does mandate this cascade, but introducing it should be a deliberate, separately-documented change, so restore the previous behavior for now. This also reverts the paired ruleutils.c logic that deparsed a column's behavior against an ERROR default: that dropped an explicit ERROR ON EMPTY from a dumped view, and otherwise emitted a redundant NULL ON EMPTY. Different now: P7 changed how ON ERROR cascades in JSON_TABLE, and this patch changes key validation in the SQL/JSON constructors. Both change SQL/JSON behavior that existing queries may rely on.

## Objections a reviewer is likely to raise

### 1. Queries that return {} today start failing, with no back-patch or release-note plan

An application runs JSON_OBJECTAGG(k VALUE v ABSENT ON NULL RETURNING json) over a left join, so unmatched rows give a NULL key with a NULL value. Today the server skips such a row. After a minor or major upgrade the whole aggregate raises an error, and the user finds out in production.

The patch turns accepted inputs into errors for both constructors and aggregates (message #7), so users see a behavior change. Message #4 counts 19 previously accepted cases that are now rejected. No message says whether this goes to stable branches or gets a release-note entry. [P7] records that a change to SQL/JSON semantics should be deliberate and documented on its own.

Precedent: [Revert cascading of JSON_TABLE's ON ERROR](https://www.postgresql.org/message-id/flat/CAA-aLv7aZGSExnbjJRw8eKkoXbu34TdoKLLA2gPye3aHjO5OSA@mail.gmail.com) (Reverted, 2026-07-14)

**Question:** Is the fix meant for master only, and which release-note text tells users that ABSENT ON NULL now rejects NULL or non-scalar keys with NULL values?

<sub>fixable, level 1b, backpatchability; messages [#4](https://www.postgresql.org/message-id/tencent_BCD8AC02BF4A6B19A64CB09D718CBDE57E0A%40qq.com), [#7](https://www.postgresql.org/message-id/CAP%3DtSTAZS%2B04tDng43JdnOAvgQ-12Z52p5YaACUo%3DtvAbJN05w%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAP%3DtSTAKkF6%3DNfGHRmPB7XyWescAxxqBAxa4YwFRk_tQg_0q3Q%40mail.gmail.com); `standards-compat-1`</sub>

### 2. The chosen semantics for omitted pairs are not checked against the SQL standard

The patch documents that key restrictions apply even to omitted pairs, which makes the behavior a promise. If the standard does not check a pair that ABSENT ON NULL drops, Postgres rejects queries that other conforming systems accept. Users who port SQL/JSON code then hit errors that only Postgres raises.

In message #6 Dmitry Dolgov says one can argue that skipped pairs need no key check, and that only the current doc wording insists on it. No reply addresses this design question. The patch then changes the documentation to match the code (message #7), so the wording settles the question instead of the standard. The array, composite and json key rules are Postgres-specific, but the NULL-key case may be covered by the standard's constructor rules.

**Question:** What does the SQL standard say about a NULL key whose pair ABSENT ON NULL drops, and does the patch follow it?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#6](https://www.postgresql.org/message-id/atKHv08UIzPGHv91%40ddolgov-thinkpadt14sgen1.rmtde.csb), [#7](https://www.postgresql.org/message-id/CAP%3DtSTAZS%2B04tDng43JdnOAvgQ-12Z52p5YaACUo%3DtvAbJN05w%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Is the fix meant for master only, or also for stable branches where queries that return {} today would start to fail?
2. What does the SQL standard say about a NULL key whose pair ABSENT ON NULL drops, and does the patch follow it?
3. Which release-note text tells users that ABSENT ON NULL now rejects NULL or non-scalar keys paired with NULL values?
4. What is the reason to validate keys of omitted pairs, given the point in message #6 that skipping the check is a defensible reading?

## Touch points

- **ABI / extension API:** v2 adds a shared helper json_validate_key_category() used by json.c and jsonb.c.
- **SQL-standard conflict or non-standard syntax:** The patch changes which keys JSON_OBJECT and JSON_OBJECTAGG accept under ABSENT ON NULL, with no check against the SQL standard.
- **pg_dump / restore round-trip:** A dumped view, check constraint or generated column whose expression hits a newly rejected key can fail when restored or refreshed.
- **backpatchability:** It is tagged as a bug fix, and queries that return {} today would raise an error if it reaches stable branches.
- **maintenance burden ("who maintains this in 10 years"):** The documentation now promises key validation for omitted pairs, which future constructor changes must keep.
- **performance on the hot path:** The patch adds a key type classification for each pair that ABSENT ON NULL omits.

## Who would commit it, and when

- Owner: silent. A committer who knows the SQL/JSON constructors and the json and jsonb aggregate code would have to own it. The visible thread shows review comments but no statement from a committer on the behavior change.
- Priority: proceed; audience niche; review cost low. The patch is small and touches only json.c, jsonb.c, the sqljson tests and the documentation. It affects only queries that pair an invalid key with a NULL value under ABSENT ON NULL, and it needs a committer's decision on back-patching and the standard.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
