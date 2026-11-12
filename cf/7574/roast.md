# Design pre-review: psql: Don't offer USING INDEX after REPACK (CONCURRENTLY) or (ANALYZE)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7574](https://commitfest.postgresql.org/patch/7574/); [thread](https://www.postgresql.org/message-id/flat/CA%2BY2BmeqAVC9Vhs%2BzyXzD%2Bu3DkNF8%3DzhRs%3DiTeK5LgFvfuHa8A%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated with the two failing commands and their error messages, and the cost stays inside psql tab completion. One fixable design objection survived, maintenance-fit-1: psql would carry its own copy of a server rule about REPACK options, and nothing ties the two together. That is a judgment for a committer who knows tab-complete.in.c, together with the back-patch question. Only one message is visible and the code was not read.

Not checked: The code was not read, so the reader's handling of unusual option lists and its fit with the existing completion rules are unchecked. The libedit behaviour is untested by the author.

## Prior attempts

None found before 2026-10-31.

## Objections a reviewer is likely to raise

### 1. Tab completion copies a server-side semantic rule that can drift

A later release lets REPACK (ANALYZE) or (CONCURRENTLY) run over all tables with a clustering index. The developer changes the server check and the docs but not tab-complete.in.c. psql keeps hiding USING INDEX for a valid command, no test fails, and users find out only when they notice the completion is missing.

Most psql completion follows the grammar, but this patch encodes a rule from the server's execution checks: which options forbid the no-table form. Message #1 adds its own reader for the option list, with last-setting-wins semantics, so psql now has a second copy of option evaluation that has to match the server. Nothing ties the two copies together. A newer psql connected to an older server, or the reverse, has no server-version check that would show the rule is per release.

**Question:** What would make a later change to the server's REPACK option rules also update or break this completion, and how should psql handle servers of a different major version?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CA%2BY2BmeqAVC9Vhs%2BzyXzD%2Bu3DkNF8%3DzhRs%3DiTeK5LgFvfuHa8A%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. If a later release relaxes the server's REPACK option rules, what would make this completion get updated or make a test fail?
2. Does any other code in tab-complete.in.c already evaluate option values, or is this option list reader the first of its kind there?
3. What does the option list reader do when a value-less option is followed by an unknown option name?
4. Do the new 010_tab_completion.pl cases pass when psql is built with libedit?
5. Do other REPACK options or option combinations also lead to a completion that always fails?

## Touch points

- **psql UX conventions:** psql stops offering USING INDEX after a REPACK option list that enables CONCURRENTLY or ANALYZE.
- **backpatchability:** Message #1 asks whether to back-patch the client-side change to REL_19_STABLE.
- **maintenance burden ("who maintains this in 10 years"):** tab-complete.in.c gains a REPACK option list reader that must keep matching the server's option rules.

## Who would commit it, and when

- Owner: silent. Someone who knows psql tab completion (tab-complete.in.c) and the REPACK option rules would have to own it. No one has replied to message #1.
- Priority: proceed; audience niche; review cost low. The 7.1 KB patch touches only tab-complete.in.c and its TAP test and helps psql users who type REPACK with these options. The one surviving objection is fixable, and its worst case is a missing suggestion.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
