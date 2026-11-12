# Design pre-review: Enlarge the syslogger pipe on Linux (F_SETPIPE_SZ)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7390](https://commitfest.postgresql.org/patch/7390/); [thread](https://www.postgresql.org/message-id/flat/CAH7T-arLtnrbM5ynVAeXYxtK%3DP012xH4%2BcrCm7w0efZq7%2BRztQ%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived the panel, so nothing stops the proposal at levels 1a, 1b or 2. The problem is stated, message #1 gives measurements for statement logging and bursts, and the change adds no user-visible surface and falls back to the kernel default. Confidence is low because only one message exists, nobody has replied, and the code was not read.

Not checked: Without the code, the retry sizes, the handling of a failed or partly granted request, and any reporting of the final pipe size could not be checked. The effect of kernel per-user pipe limits also rests on the code and on testing.

## Prior attempts

None found before 2026-09-16.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is the higher p50 at 64 and 96 clients (387 to 509 us at 96) an accepted trade-off, and does it appear when the box is not oversubscribed?
2. Which smaller sizes does the retry try, and does the syslogger report the size it got or a failed request?
3. Can pipe-max-size or the per-user pipe-user-pages-soft limit make the 1 MiB request fail, or shrink other pipes of the postgres user, on common distributions?
4. Do the results hold with logs on a real disk instead of tmpfs, where the collector's write speed differs?
5. Does a 1 MiB buffer change how much log data is lost or reordered when the syslogger dies or restarts?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch adds a Linux-only F_SETPIPE_SZ path with a retry loop in syslogger.c, and leaves Windows unchanged.
- **performance on the hot path:** The pipe size changes how long logging backends block in write(); p50 rises from 387 to 509 us at 96 clients.

## Who would commit it, and when

- Owner: silent. Someone who knows the syslogger and postmaster startup code, and Linux pipe semantics, would have to own it. Nobody has replied in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is 2.5 KiB in one file with no GUC or other user-visible surface, and message #1 shows gains for statement logging, pgaudit and auto_explain workloads. The open points are the p50 trade-off and kernel limit behaviour, which a reviewer can settle quickly.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
