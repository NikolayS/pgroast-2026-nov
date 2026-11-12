# Design pre-review: Add explicit warnings about unsafe OAuth trace output for libpq

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6864](https://commitfest.postgresql.org/patch/6864/); [thread](https://www.postgresql.org/message-id/flat/CAN4CZFOUG9csxguH_%2Bcz%3DnQ8HQVKLcVvr%3DqJSbga%2Btj-3FZhLA%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated: PGOAUTHDEBUG trace output contains bearer tokens and client secrets, and the output carries no warning. The change is two stderr messages in one file, seen only by users who set the unsafe trace flag, so it costs other users nothing. Confidence is medium because the wording and the exit paths could not be checked without the code.

Not checked: The wording of the warnings, the end warning on early-abort paths and the stderr path they use could not be checked without the code. The pass at levels 1-2 is provisional on those points.

## Prior attempts

None found before 2026-04-07.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does the end warning print on every exit path, including an early abort of the flow or a failure before the first HTTP request?
2. What is the exact wording of the two warnings, and is it the same on 18 and on master?
3. Do the warnings go to the same stderr stream as the libcurl trace, so that a redirect captures them together with the secrets?
4. Is the 18 patch meant for backpatching, given that it adds new stderr output in a stable branch?

## Touch points

- **security / privileges:** The warnings concern bearer tokens and client secrets that PGOAUTHDEBUG trace prints to stderr.
- **backpatchability:** The author offers a separate patch for 18, where the trigger is PGOAUTHDEBUG=UNSAFE; the thread has no backpatch decision.

## Who would commit it, and when

- Owner: qualified committer engaged. The owner should know the libpq-oauth client flow in oauth-curl.c. According to the brief, Daniel Gustafsson supports the idea in message #4 and proposes to go ahead unless someone objects.
- Priority: proceed; audience niche; review cost low. Only users who enable unsafe OAuth trace debugging see the change, and for them it lowers the risk of sharing secrets. The patch is under 3 KiB in one file, and the brief reports support in message #4.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
