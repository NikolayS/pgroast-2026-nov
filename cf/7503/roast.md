# Design pre-review: Reject incompatible options specified with VACUUM ONLY_DATABASE_STATS

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7503](https://commitfest.postgresql.org/patch/7503/); [thread](https://www.postgresql.org/message-id/flat/466138D9-580D-4C7B-B614-3382AB093533%40gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is sound: `vacuum (ONLY_DATABASE_STATS, PARALLEL 1)` succeeds against the documented rule. Two fixable points remain. maintenance-fit-2 asks why v3 also rejects explicitly disabled options, which the current docs wording ("enabled") allows. maintenance-fit-1 notes that per-option flags accept a future option unless its author remembers to set the flag. Neither needs a different proposal.

Not checked: The code was not read, so how the parser sets the conflict flag and whether a shared struct such as VacuumParams changes are not known. Message #4 was not shown; it may already address the choice between per-option flags and a loop over option names.

## Prior attempts

- [COPY: validate option presence rather than option values](https://www.postgresql.org/message-id/flat/C1D2509E-E5D1-46B0-932C-B57AA7B963A1@gmail.com) (similar idea; Withdrawn, 2026-06-29). reason not recorded Different now: The same author proposed presence-based option validation for COPY and withdrew it; no reason is recorded. This proposal covers one VACUUM option and points to a documentation mismatch.

## Objections a reviewer is likely to raise

### 1. Rejecting explicit PROCESS_TOAST/PROCESS_MAIN reverses a deliberate tolerance

A maintenance script runs `vacuum (ONLY_DATABASE_STATS, PROCESS_TOAST false)` and works today. After the upgrade to 20 it fails with an error, and the operator finds out when a cron job or tool reports failure after the upgrade.

The current code has the comment "don't require people to turn off PROCESS_TOAST/MAIN explicitly" (quoted in message #2), so accepting these options looks intentional. The patch rejects them anyway, and also rejects explicitly disabled options that are harmless. No message gives a user report that asks for the stricter rule, and none weighs the compatibility cost or says whether vacuumdb sends such combinations. Fixing PARALLEL, INDEX_CLEANUP, TRUNCATE and BUFFER_USAGE_LIMIT does not require breaking these no-op forms.

**Question:** Why must explicitly disabled options and explicit PROCESS_TOAST/PROCESS_MAIN be rejected, when rejecting only the options that have effects would fix the reported bug?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/466138D9-580D-4C7B-B614-3382AB093533%40gmail.com), [#2](https://www.postgresql.org/message-id/e88d7d21-7694-4913-bd4c-22266b737076%40uni-muenster.de); `maintenance-fit-2`</sub>

### 2. Denylist of given options fails open for future VACUUM options

A later release adds a new VACUUM option and its author does not set the new conflict flag in the parser. Then `vacuum (ONLY_DATABASE_STATS, NEWOPT)` is accepted and silently ignored, the same bug this patch fixes. Nothing fails at build time or in the existing tests, so a user finds out when the documented rule does not hold.

The brief and commit message say the patch tracks the incompatible options that were given, which is a denylist. The allowlist loop in message #2 rejects any new option by default and fails closed. Message #4 says both approaches need the same care, but only the denylist needs every future option author to remember ONLY_DATABASE_STATS. Message #3 argues the "_given" pattern scales better, but the publicationcmds.c and foreigncmds.c cases mainly detect duplicate options and do not enforce an allowlist.

**Question:** When someone adds a VACUUM option next year without touching this check, what makes ONLY_DATABASE_STATS reject it, or which test fails?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/e88d7d21-7694-4913-bd4c-22266b737076%40uni-muenster.de), [#3](https://www.postgresql.org/message-id/897FDEBB-8721-43DE-83D0-82E068F93185%40gmail.com), [#4](https://www.postgresql.org/message-id/404203a7-977b-49b9-b158-5006323f8e4d%40uni-muenster.de); `maintenance-fit-1`</sub>

## Questions for the author

1. Why must explicitly disabled options and explicit PROCESS_TOAST or PROCESS_MAIN be rejected, when rejecting only enabled options would fix the reported cases?
2. Do vacuumdb or other known clients send any option other than VERBOSE together with ONLY_DATABASE_STATS?
3. When someone adds a VACUUM option without touching this check, what makes ONLY_DATABASE_STATS reject it, or which test fails?
4. Is the change meant for version 20 only, given that the entry is tagged Bugfix?

## Touch points

- **SQL-standard conflict or non-standard syntax:** VACUUM option lists that are accepted today, such as `(ONLY_DATABASE_STATS, PROCESS_TOAST false)`, become errors; no new syntax is added.
- **backpatchability:** The entry is tagged Bugfix with target version 20, and the patch makes previously accepted commands fail.
- **maintenance burden ("who maintains this in 10 years"):** Each future VACUUM option must set the conflict flag in the parser, or ONLY_DATABASE_STATS accepts it silently.

## Who would commit it, and when

- Owner: silent. A committer who knows the VACUUM command code and its option parsing would have to own it; none has replied in the visible thread.
- Priority: proceed; audience niche; review cost low. The patch is about 10 KiB, touches only vacuum.c, the VACUUM reference page and regression tests, and already has one reviewer. Few users combine other options with ONLY_DATABASE_STATS, and the two open points can be settled in the thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
