# Design pre-review: Assorted Win32 error handling fixes (CreateThread, pgwin32_select, pg_usleep)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7218](https://commitfest.postgresql.org/patch/7218/); [thread](https://www.postgresql.org/message-id/flat/CAH7T-apWWfEt%2BV8uLU7KXv7CMM9LoqJ1Zo2LdWmXMz%3Du-Ek9%3Dw%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived the panel, so nothing stops patches 0003 and 0004 at levels 1a, 1b or 2. Both are small, local fixes to Win32 port code with a stated problem and no new user-facing surface. Confidence is low because nobody has reviewed 0004, and the thread does not say whether elog(FATAL) is safe for every pg_usleep() caller.

Not checked: No code was read, so the 0003 array arithmetic and the exact 0004 control flow are unchecked. Whether the new FATAL can fire in a context where elog() is unsafe needs the code and the list of pg_usleep() callers.

## Prior attempts

None found before 2026-06-10.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is elog(FATAL) safe for every caller of win32 pg_usleep(), including the postmaster and code that runs in a critical section?
2. Can pg_usleep() run before error reporting is set up, or from inside error reporting, so that the new elog(FATAL) fails or recurses?
3. Why is FATAL the right response to WAIT_FAILED, compared with a log message followed by a plain Sleep() for the requested time?
4. Should 0003 and 0004 be back-patched, given that the author says neither has a reachable in-core trigger?
5. Can pgwin32_select() receive full, disjoint read and write fd sets from any extension or frontend caller, or is 0003 defensive only?

## Touch points

- **security / privileges:** Patch 0003 removes a one-element write past the end of the events array in pgwin32_select().
- **backpatchability:** The entry is tagged Bugfix, and the thread does not say whether 0003 and 0004 should be back-patched.

## Who would commit it, and when

- Owner: silent. Heikki Linnakangas committed 0001 and 0002 in message #3 and said he did not look at 0003 and 0004. They need a committer who knows the Win32 signal emulation and the pg_usleep() callers.
- Priority: proceed; audience niche; review cost low. The two remaining patches are about 1.3 KB each, touch only src/backend/port/win32, and by the author's account in message #2 have no reachable in-core trigger. 0003 is mechanical; 0004 needs a rebase and a Windows-aware reviewer for the FATAL choice.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
