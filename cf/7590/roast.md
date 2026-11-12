# Design pre-review: BUG #19689: MERGE INSERT accepts a set-returning function during PREPARE but fails at EXECUTE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7590](https://commitfest.postgresql.org/patch/7590/); [thread](https://www.postgresql.org/message-id/flat/19689-0a1b106035cd8a5b%40postgresql.org), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is not in question: the statement fails on every execution today, and the patch moves the error to parse analysis. The only surviving objection, maintenance-fit-1, is a fixable design point about where the check lives. A committer who knows parse analysis should decide between the p_hasTargetSRFs check and an expression kind that check_srf_call_placement() rejects, and should decide on back-patching.

Not checked: The code was not read, so the error text, the SQLSTATE and the exact placement of the check are unverified. The material also does not show whether a stored SQL-function body that holds such a MERGE can exist and would fail to restore.

## Prior attempts

None found before 2026-09-15.

## Objections a reviewer is likely to raise

### 1. Ad hoc flag check instead of the expression-kind mechanism for SRF placement

A later patch adds a MERGE action or moves MERGE UPDATE SET to a new transform path. It does not copy the p_hasTargetSRFs check, and nothing fails at build or test time. Users find out when PREPARE succeeds again and every EXECUTE fails.

Postgres already decides where SRFs are allowed through ParseExprKind and check_srf_call_placement(), which gives one error text per context. The patch keeps EXPR_KIND_VALUES_SINGLE, which allows SRFs, and adds a separate test in the CMD_INSERT case only (message #2). The proposal does not say whether MERGE UPDATE SET has the same gap, so the fix may cover one of two places. A committer would likely ask for a MERGE-specific expression kind, or a reason why the existing mechanism does not fit.

**Question:** Why is a p_hasTargetSRFs check in CMD_INSERT better than giving MERGE action expressions their own expression kind, and does MERGE UPDATE SET need the same fix?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/CAHewXN%3Dt2OW_gehDsso17CLgqRfMfnhzDsby7pXWpzSy_-e9Tg%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Why is a p_hasTargetSRFs check in the CMD_INSERT case better than giving MERGE insert values an expression kind that check_srf_call_placement() rejects?
2. Should the fix be back-patched to 17 and 18, where message #1 also reproduces the failure?
3. What are the new error message text and SQLSTATE, and do they match what check_srf_call_placement() reports for other contexts?
4. Does a non-prepared MERGE with such a function now report a different error message than before?
5. Do MERGE UPDATE SET actions already reject set-returning functions at parse analysis?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The patch changes which expressions parse analysis accepts in the values of a MERGE insert action.
- **backpatchability:** Message #1 reproduces the bug on 17.11 and 18.6, and the proposal does not say whether to back-patch.
- **maintenance burden ("who maintains this in 10 years"):** The check is a separate p_hasTargetSRFs test in the CMD_INSERT case of parse_merge.c, outside check_srf_call_placement().

## Who would commit it, and when

- Owner: silent. Someone who knows parse analysis and the MERGE transform code would have to own it. No committer has spoken in the two visible messages.
- Priority: proceed; audience niche; review cost low. The patch is 1.9 KB in one parser file and fixes a reported bug that makes a prepared MERGE fail on every execution. Few users write a set-returning function in MERGE insert values, and the remaining design point is small.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
