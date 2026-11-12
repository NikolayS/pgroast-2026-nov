# Design pre-review: Fix the wrong behavior of REFRESH MATERIALIZED VIEW CONCURRENTLY

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6579](https://commitfest.postgresql.org/patch/6579/); [thread](https://www.postgresql.org/message-id/flat/40d694df-39fd-4a4a-9459-9d6489165f60%40gogi.tv), 22 of 22 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: the thread agrees that the concurrent refresh can leave a view that differs from its query. The two surviving objections, maintenance-fit-1 and maintenance-fit-2, are fixable design points. The header comment still states the wrong invariant, and the patch has not been compared with the explicit NULL check from message #21. Confidence is medium because only 3 of 22 messages and the first patch's prose were visible.

Not checked: The code and the later patch versions were not read, so the exact pre-check query, its schema qualification of operators and its cost could not be checked. Only 3 of 22 messages were visible in full.

## Prior attempts

None found before 2026-02-09.

## Objections a reviewer is likely to raise

### 1. Fix leaves the documented invariant of refresh_by_match_merge() wrong

A later developer works on the issue 2 slowdown or another change to the diff join and trusts the header comment saying the join is correct for any row with a NULL. They change which key columns enter the join or the pre-check. Duplicates with NULLs outside key columns get through again, and no test or message catches it until a user reports a stale view.

In message #21 Robert Haas says the header comment is wrong in two ways and that the fix should update it. In message #22 Yugo Nagata asks for the comment to explain why the NULL-in-key-column case works. The only patch prose shown, and v7-0001 as described, leave that comment as it is. The correctness of this code rests on a subtle interaction between *= and per-column equality, so the comment is the main defence against regressions here.

**Question:** Can the next version rewrite the header comment to state the real invariant, NULLs in unique-index columns only, and say why the join is then correct?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#21](https://www.postgresql.org/message-id/CA%2BTgmoaryPLAMNWhF8cP4L%2Bf%2BwQKkwS5OGCC4rUCwfpm%2Bu9-fg%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/20260731164448.f84babf2a951ad20e68058a8%40sraoss.co.jp); `maintenance-fit-1`</sub>

### 2. Pre-check mechanism chosen without comparing it to explicit NULL checks

After a minor release, a site with a large concurrently refreshed matview whose rows often hold NULLs in non-key columns finds the refresh slower. Those rows were skipped by the old IS NOT NULL filter but now enter the EXISTS self-join on newdata. Nobody measured this before the release.

Robert Haas (message #21) suggested a narrower change that checks only for NULLs in unique-index columns. Yugo Nagata (message #22) asked why the patch uses equality operators instead of explicit NULL checks, and the thread has no answer. The thread shows no timing for the new pre-check on large views, only regression-test passes (message #20). A back-patched bug fix should pick the variant that is easiest to reason about and show what it costs.

**Question:** How does the equality-operator pre-check compare with Robert's explicit NULL check in clarity and in refresh time on a large view with many NULLs in non-key columns?

<sub>fixable, level 2, design; messages [#20](https://www.postgresql.org/message-id/CAOVWO5pSxK9TZEEQ7xgNQOUFob_9F-u3YRRogXjDyRktgfaC%2BQ%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CA%2BTgmoaryPLAMNWhF8cP4L%2Bf%2BwQKkwS5OGCC4rUCwfpm%2Bu9-fg%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/20260731164448.f84babf2a951ad20e68058a8%40sraoss.co.jp); `maintenance-fit-2`</sub>

## Questions for the author

1. Can the next version rewrite the header comment of refresh_by_match_merge() to state the real invariant and explain why the join is correct for NULLs in unique-index columns?
2. Why does the pre-check use the per-column equality operators instead of the explicit NULL check on unique-index columns suggested in message #21?
3. How long does the new pre-check take on a large view with many NULLs in non-indexed columns, compared with the current code?
4. Is the fix meant for back-patching, and if so, what should the release note tell users whose refreshes will start to fail?
5. Which patch version does CommitFest entry 6579 track now, and does its commit message still claim the (NULL, NULL) case as a fix?

## Touch points

- **backpatchability:** A back-patched fix makes some concurrent refreshes that succeed today fail with the duplicate-rows error after a minor upgrade.
- **maintenance burden ("who maintains this in 10 years"):** The header comment of refresh_by_match_merge() states an invariant that the fix shows to be wrong, and the patch leaves it unchanged.
- **performance on the hot path:** The changed pre-check runs on every concurrent refresh and now includes rows with NULLs; the thread shows no timing.

## Who would commit it, and when

- Owner: qualified committer engaged. Robert Haas reviewed the fix in message #21 and asked for the header comment to be rewritten; the owner needs to know the matview diff join and row comparison operators.
- Priority: proceed; audience a sizeable minority; review cost low. This is a wrong-result fix in one function of matview.c with no new user-visible surface, and reviewers agree it is a bug. It needs a new version that answers messages #21 and #22 before a committer can take it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
